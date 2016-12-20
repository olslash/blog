I came across this problem online:

    Find a 9 letter string of characters that contains only letters from {acdegilmnoprstuw} such that the hash(the_string) is 945924806726376.

You're given a hashing function, which I translated to Python as follows:
```python
    def hsh(s):
      h = 7
      letters = "acdegilmnoprstuw"
      for i, val in enumerate(s):
        h = (h * 37 + letters.index(s[i]))
        return h
```
        
It could be solved in a number of ways, but I decided to constrain myself to a solution that treats the hashing function as a black box. That is, only exploiting properties that could be easily observed from the outside.

My solution came first from the observation that letters found in later indices in the 'potential characters' list created higher resulting hashed values. For example, a string of 9 'w' characters evalutates to the highest possible hash for length = 9, because w is the last character in the list.

Because of this, if a potential partial answer ('ac') is padded with 'w's until the total length of the input is 9 ('acwwwwwww'), the hash of that input has to be greater than the target hash for the partial answer to be valid. For example, if I think the first letter might be 'd', I pass 'dwwwwwwww' into the hashing function, with the result 918220670579181. That's less than the target hash, so 'd' is not a possible choice for the first character in the result. The character 's' padded with 'w's is 953345465118391; greater than the target, so the branch starting with 's' is worth exploring.

The algorithm, then, recursively tries each possible character from the available options up to the target length, immediately pruning branches that start with impossible characters– that is, characters that would prevent the hash from ever going high enough to reach its target value.

<script src="https://gist.github.com/olslash/b2f93144bcec707310b9.js"></script>

The algorithm would of course work without pruning, but the time needed to calculate every possibility up to nine characters is huge. By cutting many possible branches out early, we're able to effectively find that answer is 'promenade'.
