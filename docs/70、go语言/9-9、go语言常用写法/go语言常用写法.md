## 字符串

* strings.ToLower(s[0:1])
* strings.IndexRune(s, '@')
* regexp.MustCompile(`\{|\}|\+|\-|  | `)
* pattern.ReplaceAllString(s, "")

* 遍历字符串字符(assi)

  直接遍历字符串，字符串是不可变数组

  ```
  board []string
  m := len(board)
  for i := 0; i < m; i++ {
  	for j := 0; j < m; j++ {
  		if board[i][j] == 'X' || board[i][j] == ' ' {
  				rowX++
  		}
  	}
  }
  ```

  

  board []string