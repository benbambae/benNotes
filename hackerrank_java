# Java HackerRank Guidebook

---

## 1. Collections & Data Structures

### Array
**When to use:** Fixed-size collection, need index access, memory efficiency matters, size known at compile time
**Question keywords:** `fixed size`, `n elements`, `given array of size`, `index access`, `constant time lookup by position`

```java
int[] a = new int[n];          // declare
a[i] = value;                  // assign
a[0]; a[i];                    // access
a.length;                      // length

for (int x : a) { }            // enhanced loop
```

### ArrayList
**When to use:** Dynamic size needed, frequent insertions/removals, don't know size upfront
**Question keywords:** `dynamic`, `growing list`, `add elements`, `remove elements`, `unknown number of items`

```java
ArrayList<Integer> list = new ArrayList<>();

list.add(10);                  // append
list.add(1, 20);               // insert at index
int x = list.get(0);           // read
list.set(0, 99);               // replace
list.remove(0);                // remove by index
list.size();                   // length
```

### Stack
**When to use:** Last-In-First-Out (LIFO), matching pairs, undo operations, backtracking
**Question keywords:** `balanced`, `parentheses`, `brackets`, `valid`, `matching`, `nested`, `undo`, `reverse order`, `backtrack`

```java
Stack<Character> stack = new Stack<>();

stack.push('a');               // add to top
stack.pop();                   // remove from top
stack.peek();                  // see top (no remove)
stack.isEmpty();               // check empty
```

**Bracket Pattern (VERY COMMON)**
```java
Stack<Character> stack = new Stack<>();
for (char c : s.toCharArray()) {
    if (c == '(' || c == '{' || c == '[') {
        stack.push(c);
    } else {
        if (stack.isEmpty()) return false;
        char top = stack.pop();
        if ((c == ')' && top != '(') ||
            (c == '}' && top != '{') ||
            (c == ']' && top != '[')) return false;
    }
}
return stack.isEmpty();
```

### Queue / Deque
**When to use:** First-In-First-Out (FIFO), processing in order, BFS traversal, sliding window
**Question keywords:** `first come first serve`, `process in order`, `BFS`, `level order`, `sliding window`, `stream of data`

```java
Queue<Integer> q = new LinkedList<>();
q.add(1);                      // enqueue
q.poll();                      // dequeue
q.peek();                      // front
q.isEmpty();

Deque<Integer> dq = new ArrayDeque<>();
dq.addLast(1);
dq.removeFirst();
dq.peekFirst();
```

---

## 2. Maps & Sets

### HashMap
**When to use:** Key-value lookup, counting frequency, grouping by property, O(1) lookup by key
**Question keywords:** `count occurrences`, `frequency`, `how many times`, `group by`, `lookup`, `mapping`, `pairs`, `anagram`

```java
Map<String, Integer> map = new HashMap<>();

map.put(key, value);           // insert/update
map.get(key);                  // retrieve
map.containsKey(key);          // check exists
map.getOrDefault(key, 0);      // get with fallback
```

**Frequency Count (VERY COMMON)**
```java
map.put(k, map.getOrDefault(k, 0) + 1);
```

**Iteration**
```java
for (String key : map.keySet()) {
    System.out.println(key + " " + map.get(key));
}
```

### HashSet
**When to use:** Track unique values, detect duplicates, O(1) membership check
**Question keywords:** `unique`, `distinct`, `duplicate`, `already seen`, `exists`, `first repeating`, `no duplicates`

```java
Set<String> set = new HashSet<>();

set.add(item);                 // add
set.contains(item);            // check exists
set.size();                    // count unique
```

**Remove Duplicates**
```java
Set<Integer> set = new HashSet<>();
for (int x : arr) set.add(x);
```

**Unique Pairs (COMMON)**
```java
Set<String> set = new HashSet<>();
String pair = left + " " + right;
set.add(pair);
```

---

## 3. Strings

### String Basics

```java
s.length();                    // length
s.charAt(i);                   // char at index
s.substring(1, 4);             // slice [1,4)
s.equals(t);                   // compare
s.split(" ");                  // split to array
char[] c = s.toCharArray();    // to char array
s.toLowerCase();               // to lowercase
s.toUpperCase();               // to uppercase
Integer.parseInt(s);           // string to int
```

**Reverse a String**

```java
String reversed = new StringBuilder(s).reverse().toString();
```

### Character Methods

```java
Character.isDigit(c);          // '0'-'9'
Character.isLetter(c);         // a-z, A-Z
Character.isUpperCase(c);
Character.isLowerCase(c);
```

### StringBuilder (USE IN LOOPS)
**When to use:** Building strings in loops, concatenating many strings, string manipulation
**Question keywords:** `build string`, `concatenate in loop`, `construct output`, `append characters`

```java
StringBuilder sb = new StringBuilder();
sb.append("a");
sb.append("b");
String s = sb.toString();

// NEVER do: s = s + "a"; in loops (slow)
```

---

## 4. Common Patterns

### Two Pointers
**When to use:** Sorted array operations, palindrome check, pair finding, in-place modifications
**Question keywords:** `palindrome`, `reverse`, `sorted array`, `pair with sum`, `two sum sorted`, `in-place`, `opposite ends`

```java
int l = 0, r = n - 1;
while (l < r) {
    // process
    l++;
    r--;
}
```

Used for: reverse, palindrome, sorted array search

### Prefix Sum
**When to use:** Range sum queries, subarray sums, multiple sum queries on same array
**Question keywords:** `range sum`, `subarray sum`, `sum between`, `cumulative`, `multiple queries`, `sum of elements from i to j`

```java
int[] prefix = new int[n + 1];
for (int i = 0; i < n; i++) {
    prefix[i + 1] = prefix[i] + a[i];
}

// Range sum [l, r)
int sum = prefix[r] - prefix[l];
```

### Sorting

```java
Arrays.sort(arr);                          // primitive array
Collections.sort(list);                    // ArrayList ascending
Collections.sort(list, (a, b) -> b - a);   // ArrayList descending
Arrays.sort(arr, (a, b) -> a[0] - b[0]);   // 2D by first element
```

---

## 5. Essentials

### Scanner Setup

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
String s = sc.nextLine();
sc.close();
```

### Input Bug Fix (COMMON)

```java
int n = sc.nextInt();
sc.nextLine();                 // ALWAYS consume newline after nextInt()
String s = sc.nextLine();
```

### Math

```java
Math.max(a, b);
Math.min(a, b);
Math.abs(x);
Math.pow(2, n);
n % 2 == 0                     // even check
n % k == 0                     // divisible by k
```

### Boolean Flag

```java
boolean found = false;
// ... loop ...
if (condition) found = true;
```

---

## Quick Reference

| Structure | Create | Add | Get | Size |
|-----------|--------|-----|-----|------|
| Array | `int[] a = new int[n]` | `a[i] = x` | `a[i]` | `a.length` |
| ArrayList | `new ArrayList<>()` | `add(x)` | `get(i)` | `size()` |
| HashMap | `new HashMap<>()` | `put(k,v)` | `get(k)` | `size()` |
| HashSet | `new HashSet<>()` | `add(x)` | `contains(x)` | `size()` |
| Stack | `new Stack<>()` | `push(x)` | `peek()` | `isEmpty()` |
| Queue | `new LinkedList<>()` | `add(x)` | `peek()` | `isEmpty()` |

---

## Input/Output Patterns

### Single Test Case, N Items (Meaning just ONE Output)
**When to use:** Simple problem with one test case, process N items sequentially
**Question keywords:** `given n`, `for each element`, `read n integers`, `process array`

```java
int n = sc.nextInt();
for (int i = 0; i < n; i++) {
    int x = sc.nextInt();
    // process
}
```

### Multiple Test Cases (T Cases)
**When to use:** Problem has multiple independent test cases, each needs separate output
**Question keywords:** `t test cases`, `for each test case`, `number of queries`, `repeat t times`

```java
int t = sc.nextInt();
while (t-- > 0) {
    int n = sc.nextInt();
    // process each case
    System.out.println(result);
}
```

### Read Until EOF
**When to use:** Unknown number of inputs, read until end of file
**Question keywords:** `read until end`, `unknown number of lines`, `process all input`, `until EOF`

```java
while (sc.hasNext()) {
    String s = sc.next();
    // process
}
```

### Matrix / 2D Input
**When to use:** Grid-based problems, 2D array input, matrix operations
**Question keywords:** `n x m grid`, `matrix`, `2D array`, `rows and columns`, `grid traversal`

```java
int n = sc.nextInt();
int m = sc.nextInt();
int[][] grid = new int[n][m];
for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        grid[i][j] = sc.nextInt();
    }
}
```

### Output Per Query (print inside loop)
**When to use:** Each query needs immediate output, results are independent
**Question keywords:** `print for each query`, `output per line`, `answer each query`, `respond immediately`

```java
int q = sc.nextInt();
while (q-- > 0) {
    // read query
    System.out.println(answer);  // print immediately
}
```

### Collect Then Output (print at end)
**When to use:** Performance matters, many outputs, batch printing is faster
**Question keywords:** `large output`, `many lines`, `performance critical`, `batch output`

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) {
    sb.append(result).append("\n");
}
System.out.print(sb);  // single print at end (faster)
```

---

## Sample I/O Templates

### 1. N + Mixed Data Per Line (VERY COMMON)
**When to use:** Each line has multiple fields (ID, name, score, etc.)
**Question keywords:** `n students`, `n records`, `id name score`, `multiple fields per line`

```
Sample Input:
5
33 Rumpa 3.68
85 Ashis 3.85
56 Samiha 3.75
19 Samara 3.75
22 Fahim 3.76

Sample Output:
Ashis
Fahim
Samara
Samiha
Rumpa
```

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();

for (int i = 0; i < n; i++) {
    int id = sc.nextInt();
    String name = sc.next();       // next() for single word (no spaces)
    double score = sc.nextDouble();
    // process...
}
sc.close();
```

---

### 2. N + One Value Per Line
**When to use:** First line is count, then one value per line
**Question keywords:** `n numbers`, `one per line`, `n integers`

```
Sample Input:
4
5
10
3
7

Sample Output:
25
```

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();

for (int i = 0; i < n; i++) {
    int x = sc.nextInt();
    // process...
}
sc.close();
```

---

### 3. N + Space-Separated on One Line
**When to use:** Count on first line, all values on second line
**Question keywords:** `n space-separated integers`, `array on one line`

```
Sample Input:
5
3 9 1 7 4

Sample Output:
1 9
```

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
int min = Integer.MAX_VALUE;
int max = Integer.MIN_VALUE;

for (int i = 0; i < n; i++) {
    int x = sc.nextInt();
    min = Math.min(min,x);
    max = Math.max(max,x);
}
System.out.println(min+" "+max);
sc.close();
```

Another Example: Reverse Output

```
Sample Input:
5
10 20 30 40 50

Sample Output:
50 40 30 20 10
```

```java
int n = sc.nextInt();
int[] arr = new int[n];

for (int i = 0; i < n; i++) {
    arr[i] = sc.nextInt();
}

for (int i = n - 1; i >= 0; i--) {
    System.out.print(arr[i] + " ");
}

```

Another Example: Check if sorted

```
Sample Input:
5
1 2 3 5 4

Sample Output:
NO
```

```java
int n = sc.nextInt();
boolean sorted = true;

int prev = sc.nextInt();
for (int i = 1; i < n; i++) {
    int cur = sc.nextInt();
    if (cur < prev) sorted = false;
    prev = cur;
}

System.out.println(sorted ? "YES" : "NO");

```



---

### 4. Single Line Space-Separated (No Count Given)
**When to use:** Just one line of values, no count provided
**Question keywords:** `space-separated`, `split`, `words on a line`

```
Sample Input:
hello world foo bar

Sample Output:
4
```

```java
Scanner sc = new Scanner(System.in);
String line = sc.nextLine();
String[] parts = line.split(" ");

for (String word : parts) {
    // process each word...
}
sc.close();
```

**TRAP:** If you previously used `nextInt()`/`next()`, then `nextLine()` reads the leftover newline (empty string)!
```java
// Safe pattern after nextInt()/next():
sc.nextLine();                     // consume leftover newline first
String line = sc.nextLine();       // now reads actual content
String[] parts = line.split(" ");
```

---

### 5. Two Numbers Then Grid/Pairs
**When to use:** Dimensions first (N M), then grid or pair data
**Question keywords:** `n x m`, `rows columns`, `n rows m columns`, `grid`

```
Sample Input:
3 4
1 2 3 4
5 6 7 8
9 10 11 12

Sample Output:
78
```

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
int m = sc.nextInt();
int[][] grid = new int[n][m];

for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        grid[i][j] = sc.nextInt();
    }
}
sc.close();
```

---

### 6. N + Full Line Strings (WATCH FOR BUG)
**When to use:** Reading complete lines with spaces after reading a number
**Question keywords:** `n strings`, `full line`, `sentence per line`

```
Sample Input:
3
Hello World
Foo Bar Baz
Test Line

Sample Output:
Hello World
Foo Bar Baz
Test Line
```

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
sc.nextLine();                     // CRITICAL: consume leftover newline!

for (int i = 0; i < n; i++) {
    String line = sc.nextLine();   // now reads full line correctly
    // process...
}
sc.close();
```

**WARNING:** Forgetting `sc.nextLine()` after `nextInt()` is the #1 input bug!

---

### 7. Q Queries with Multiple Values Each
**When to use:** Number of queries, then each query has parameters
**Question keywords:** `q queries`, `query type`, `operations`, `commands`

```
Sample Input:
3
1 5 10
2 3
1 2 7

Sample Output:
15
3
9
```

```java
Scanner sc = new Scanner(System.in);
int q = sc.nextInt();

while (q-- > 0) {
    int type = sc.nextInt();
    int result;

    if (type == 1) {
        int a = sc.nextInt(), b = sc.nextInt();
        result = a + b;
    } else {
        int x = sc.nextInt();
        result = x;
    }
    System.out.println(result);
}
sc.close();
```

---

### 8. Read Until EOF (VERY COMMON)
**When to use:** Unknown number of inputs, no count given, read everything
**Question keywords:** `read until end`, `process all input`, `phone book`, `unknown number of lines`

```
Sample Input:
sam 99912222
tom 11122222
harry 12299933
sam
edward
harry

Sample Output:
sam=99912222
Not found
harry=12299933
```

```java
Scanner sc = new Scanner(System.in);

while (sc.hasNext()) {             // or hasNextInt(), hasNextLine()
    String s = sc.next();
    // process...
}
sc.close();
```

**Variants:**

- `sc.hasNext()` — any token available
- `sc.hasNextInt()` — integer available
- `sc.hasNextLine()` — line available

---

### Quick Scanner Method Reference

| Data Type | Method | Reads |
|-----------|--------|-------|
| `int` | `sc.nextInt()` | Next integer |
| `long` | `sc.nextLong()` | Next long |
| `double` | `sc.nextDouble()` | Next decimal |
| `String` (word) | `sc.next()` | Next word (stops at space) |
| `String` (line) | `sc.nextLine()` | Entire line (including spaces) |

**Golden Rule:** After `nextInt()`/`nextDouble()`/`next()`, if you need `nextLine()`, call `sc.nextLine()` once to consume the leftover newline first!
