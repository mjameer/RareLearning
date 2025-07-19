```
class Solution:
    def groupAnagrams(self, strs: List[str]) -> List[List[str]]:
        anagram_map = defaultdict(list)
        for word in strs :
            key = ''.join(sorted(word))
            anagram_map[key].append(word)

        return list(anagram_map.values())
```


```
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> anagramsMap = new HashMap<>();

        for (String word : strs) {

            char [] charArray = word.toCharArray();
            Arrays.sort(charArray);
            String key = new String(charArray); 

            anagramsMap.computeIfAbsent(key, k -> new ArrayList<String>()).add(word);
        }

        return new ArrayList<>(anagramsMap.values());
        
    }
}
```
