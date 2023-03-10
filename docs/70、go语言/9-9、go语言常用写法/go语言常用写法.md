## 字符串

* strings.ToLower(s[0:1])

* !unicode.IsLower(rune(s[i - 1]))

  判断是否为小写

*  unicode.IsDigit(v)

* strings.ContainsRune(".!,", v)

  v为rune类型 ，是否包含

* strings.Fields(sentence)

  按空格切割字符串

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

## 优先队列

## treemap

```
type MyCalendarThree struct {
    *redblacktree.Tree
}


func Constructor() MyCalendarThree {
    return MyCalendarThree{redblacktree.NewWithIntComparator()}
}

func (t MyCalendarThree) add(x, val int) {
   if v, e := t.Get(x); e {
       val += v.(int)
   }
   t.Put(x, val)
}

func (t MyCalendarThree) Book(startTime int, endTime int) (ans int) {
    t.add(startTime, 1)
    t.add(endTime, -1)
    sum := 0
    for t := t.Iterator();t.Next();{
         sum += t.Value().(int)
        if sum > ans {
            ans = sum
        }
    }
    return
}


/**
 * Your MyCalendarThree object will be instantiated and called as such:
 * obj := Constructor();
 * param_1 := obj.Book(startTime,endTime);
 */
```

