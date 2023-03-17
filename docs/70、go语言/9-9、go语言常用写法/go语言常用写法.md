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

## 排序

* sort.Search

  ```
   var m = len(a)
   f := sort.Search(m-1, func(y int) bool {
   	return maxInts(a[y]) > maxInts(a[y+1])
   })
  return []int{f, maxIntsIdx(a[f])}
  ```

  search使用二分法进行查找，Search()方法回使用“二分查找”算法来搜索某指定切片[0:n]，并返回能够使f(i)=true的最小的i（0<=i<n）值，并且会假定，如果f(i)=true，则f(i+1)=true，即对于切片[0:n]，i之前的切片元素会使f()函数返回false，i及i之后的元素会使f()函数返回true。但是，当在切片中无法找到时f(i)=true的i时（此时切片元素都不能使f()函数返回true），Search()方法会返回n（而不是返回-1）。

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

