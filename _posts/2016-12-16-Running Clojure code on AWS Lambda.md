For a little while now I've been messing around with Clojure and Lambda as components of a [side project](https://github.com/olslash/bag). The basic idea for this little piece is that I get a bunch of image URLs along with some metadata from an api like Imgur, then send those URLs to Lambda which reuploads them into an S3 bucket. 

Lambda's nice here because I can keep the instance that runs my backend process lightweight and avoid the bandwidth/memory costs of transferring potentially thousands of images in parallel.

The thing that took me probably the most time was just getting my Lambda code to execute properly on AWS. Since Lambda doesn't support Clojure directly, I had to compile a Java JAR file, which led to a lot of headaches as I learned how to make everying work. Additionally, this was my first time really digging into the docs for a Java API, so I learned a lot about interop.


### The lambda function
The function will accept an input stream over which a JSON payload describing the work to be done will be received, and a [lambda context object](http://docs.aws.amazon.com/lambda/latest/dg/java-context-object.html). It will write its response to an [output stream](http://docs.aws.amazon.com/lambda/latest/dg/java-context-object.html). 

The first piece of code abstracts away the stream details and runs the actual handler as a callback (we assume that the handler will return a promise or future that can be deref'd with `@`).

```clojure
;; ns dependeicies
(:require [cheshire.core :as json]
          [clojure.java.io :as io])

(defn lambda-handler [in out ctx handler-fn]
  ;; `true` keywordizes object keys.
  (let [event (json/parse-stream (io/reader in) true) 
        res   @(handler-fn event ctx)]
    (with-open [w (io/writer out)]
      (json/generate-stream res w))))

```

Now we can define the handler function itself. I used the excellent [Lambada library](https://github.com/uswitch/lambada), which makes sure the proper Java classes will be generated for Lambda to invoke.

```clojure
(deflambdafn lambda.my-handler.handler [in out ctx]
  (lambda-handler in out ctx handle-event))
```

And the actual handler looks like this (this is the function body Lambda will run):

```clojure
(defn handle-event
  ;; destructure the JSON payload sent from our server
  [{:keys [image-url             ; the image to fetch
           image-fetch-headers   ; headers (like auth) needed to fetch
           bucket-name           ; the s3 bucket to upload to
           file-name             ; the image filename in s3
           folder-name           ; the image folder in s3
           image-meta]}          ; map of metadata like attribution info,
                                 ; description, nsfw?, etc
   ctx] ; unused here, but has meta about the Lambda environment
  
  ;; fetch the image with http-kit (blocking)
  (let [image           (:body @(http/get image-url {:headers (or image-fetch-headers {})
                                                     :as      :stream}))
        ;; S3 is picky about filenames, so i strip most special characters.
        file-path       (str folder-name "/" (str/replace file-name #"[^a-zA-Z \.\d-_]" ""))

        ;; invoke the upload-s3-file helper to upload the image
        s3-upload       (upload-s3-file bucket-name file-path image (or sanitized-meta {}))
        ;; pull out the function to add a progress listener to the upload 
        ;; (see TransferManager in the AWS Java SDK; s3-upload is a map with a few other useful keys)
        add-listener-fn (:add-progress-listener s3-upload
        out             (promise))]

    ;; set a callback that will deliver the result of the upload to the `out` promise.
    ;; :upload-result has the key the file was saved at and some other useful meta
    (add-listener-fn

      ;; Lambda uses the statusCode, headers and body fields to construct its http response to our app.
      ;; This example shows the happy path, but error handling can and should take into account
      ;; the variety of errors that can be thrown from the http fetch, s3 upload, etc.

      #(when (= :completed (:event %)     
                (deliver out {:statusCode 200
                              :headers    {}
                              :body       {:ok     true
                                           :result ((:upload-result s3-upload))}}))))

    ;; return the promise
    out))
```
Where the `upload-s3-file` is a simple helper that calls `upload` from the [Amazonica](https://github.com/mcohen01/amazonica) library, a wrapper for the AWS Java SDK.

```clojure
;; ns dependencies:
(:require [amazonica.aws.s3transfer :refer [upload]])
  
(defn upload-s3-file
  ([bucket-name file-name input-stream user-metadata]
   (let [content-length (.available input-stream)]
     (upload {:endpoint      "us-west-1"
              ;; Some tweaking is probably needed here. Before I turned up these timeouts, 
              ;; I was seeing lots of hard-to-debug errors from Lambda.
              :client-config {:max-error-retry          10
                              :socket-timeout           20000
                              :connection-timeout       20000
                              :request-timeout          20000
                              :client-execution-timeout 20000}}
             bucket-name
             file-name
             input-stream
             ;; Tell S3 the length of the image stream we're going to send it,
             ;; along with our image metadata
             {:content-length content-length
              :user-metadata  user-metadata})))
  ;; default meta to empty map
  ([bucket-name file-name input-stream] (upload-s3-file bucket-name file-name input-stream {})))
```

### Invoking the function

Again, just a simple wrapper around an Amazonica function

```clojure
;; ns dependencies
(:require [amazonica.aws.lambda :refer [invoke]])
(:import (java.nio.charset StandardCharsets))

(defn invoke-lambda-fn [fn-name payload]
  (let [res  (invoke {:endpoint "us-west-1"}
                     :function-name fn-name
                     :payload (json/generate-string payload))
        ;; Convert response buffer to string, then parse as JSON.
        ;; This receives the response we created in `handle-event` above
        json (-> (String. (.array (:payload res)) StandardCharsets/UTF_8)
                 (json/parse-string true))
        ;; happy path, add error handling here;
        ;; can be :errorMessage instead of :body
        :body
        :result]
    json))
```

### Generating an Uberjar and uploading it to Lambda

```clojure
;; In project.clj

;; The lambda and backend code are in different namespaces. 
;; We generate two uberjars, one for each of the profiles below,
;; and send lambda.jar to AWS.
:profiles { :lambda  {:dependencies []
                      :uberjar-name "lambda.jar"
                      :main         lambda.core
                      :aot          [lambda.core]}
            :core    {:main         ^:skip-aot project.core
                      :uberjar-name "main.jar"}}

```

To generate uberjars, run:

`lein clean; lein with-profile lambda uberjar`
and
`lein clean; lein with-profile core uberjar`

I wrote a [bash script](https://github.com/olslash/bag/blob/master/script/refresh-lambda-scripts.sh) and associated [config file](https://github.com/olslash/bag/blob/master/script/lambda_management_config.conf) to make it easy to generate the jar and upload it to AWS in one step. If we were to add more lambda functions to the namespace, the script would use the same jar for each one, just specifying a different handler function.

#### Note: Keeping the bundle size down
Because Lambda has a max size for uploaded code, we want to avoid including the entire AWS Java SDK, but the Amazonica depdency makes that happen by default. To avoid it, exclude the java SDK from amazonica, then explicitly include the pieces you need as separate deps:

```clojure
[amazonica "0.3.77" :exclusions [com.amazonaws/aws-java-sdk
                                com.amazonaws/amazon-kinesis-client]]
[com.amazonaws/aws-java-sdk-core "1.11.63"]
[com.amazonaws/aws-java-sdk-lambda "1.11.63"]
[com.amazonaws/aws-java-sdk-s3 "1.11.63"]]
```

### Other notes
Startup time is an issue. If many invocations of the same Lambda function are made within ~a few seconds to a minute, the container (or something) is reused, so the startup time is greatly reduced, but the first time you invoke a function it can take up to 10 seconds to actually run.

A possible solution is using Clojurescript to compile to JS, which can be run in Node. The Node SDK would have to be used, so a lot of this code wouldn't work anymore.
