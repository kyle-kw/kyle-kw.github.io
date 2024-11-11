+++
title = 'SICP系列-流处理'
date = 2024-01-11T11:02:17+08:00
tags = ["SICP", "Lisp"]
+++

本文总结一下SICP中流处理。

### 为什么需要流？

第一次接触流，我就有了下面的疑问。为什么引入流，看起来和cons、car、cdr的作用一样？

当某一些问题形式类似，就可以将他们抽象处理。比如处理前n项fib奇数和，以及树中奇数平方和等等。这类问题都是遍历（map）一个结构，筛选（filter）数据，合并数据。所以可以找到一种通用的流程来处理这类问题。

理论上list也可以处理这种问题，为什么还要引入流？那是因为list数据结构会生成所有数据，而流是懒加载的，只有当使用时才会加载数据。理论上可以使用流构建一个无限数组。
```lisp
(define (integer n)
  (cons-stream n (integer (+ n 1))))

(define infinite (integer 1))
```
可以根据这个特性，在小内存中处理大量的数据。


### 流的本质是什么？怎么使用流？
lisp中流的语法
```lisp
(cons-stream x y)
(head s)
(tail s)
the-empty-stream

(head (cons-stream x y)) ; -> x
(tail (cons-stream x y)) ; -> y
```
根据lisp语法，自己实现流。
```lisp
(define (const-stream x y) (cons x (delay y)))
(define (head s) (car x))
(define (tail s) (force (cdr s)))

(define (deley exp) (memo-proc (lambda() exp)))
; memo-proc 缓存上一次计算的结果，防止重复计算
(define (force exp) (exp))

(define (memo-proc proc)
  (let ((already-run? false)
        (result false))
    (lambda ()
      (if (not already-run?)
          (begin (set! result (proc))
                 (set! already-run? true)
                 result)
          result))))
```

使用`memo-proc`主要避免两个问题
1. 重复计算
```lisp
(define ones (cons-stream 1 ones))

(tail ones)       ; 创建新的 (cons 1 (delay ones))
(tail (tail ones)); 再次创建新的 (cons 1 (delay ones))
```

2. 可能导致无限循环
```lisp
(define fibs
  (cons-stream 0
    (cons-stream 1
      (stream-map + fibs (stream-cdr fibs)))))

; 每次访问 (stream-cdr fibs) 都会重新计算
; 计算过程会不断递归调用自身, 最终导致无限循环或栈溢出
```

### 有哪些最佳实践？
八皇后问题是一个经典的回溯算法问题，目标是在8×8的国际象棋棋盘上放置8个皇后，使得任意两个皇后都不能互相攻击。如果使用回溯需要处理很多种情况，判断当前位置是否冲突、是否回溯、记录当前状态。对于lisp来说比较复杂。

可以按照“愿望”编程，将某一个部分视为一个整体。假设k-1的问题已经解决，只需要考虑第k情况怎么处理就可以了。需要注意递归处理的边界。

但是又因为递归导致空间复杂度指数增长，因此可以使用流，来处理这种情况。
```lisp
; 首先定义基本的流操作
(define-syntax cons-stream
  (syntax-rules ()
    ((cons-stream head tail)
     (cons head (delay tail)))))

(define (stream-car stream) (car stream))
(define (stream-cdr stream) (force (cdr stream)))

(define stream-null? null?)
(define the-empty-stream '())

(define (stream-enumerate-interval low high)
  (if (> low high)
      the-empty-stream
      (cons-stream low (stream-enumerate-interval (+ low 1) high))))

(define (stream-filter pred stream)
  (cond ((stream-null? stream) the-empty-stream)
        ((pred (stream-car stream))
         (cons-stream (stream-car stream)
                     (stream-filter pred (stream-cdr stream))))
        (else (stream-filter pred (stream-cdr stream)))))

(define (stream-map proc . argstreams)
  (if (stream-null? (car argstreams))
      the-empty-stream
      (cons-stream
       (apply proc (map stream-car argstreams))
       (apply stream-map
              (cons proc (map stream-cdr argstreams))))))

(define (stream-append s1 s2)
  (if (stream-null? s1)
      s2
      (cons-stream (stream-car s1)
                  (stream-append (stream-cdr s1) s2))))

; 检查是否安全的函数
(define (safe? positions)
  (let ((queen-row (car positions)))
    (define (iter rest-queens i)
      (or (null? rest-queens)
          (and (not (= queen-row (car rest-queens)))           ; 检查同行
               (not (= queen-row (+ (car rest-queens) i)))     ; 检查对角线
               (not (= queen-row (- (car rest-queens) i)))     ; 检查对角线
               (iter (cdr rest-queens) (+ i 1)))))
    (iter (cdr positions) 1)))

; 生成下一列的所有可能位置
(define (adjoin-position row rest-queens)
  (cons row rest-queens))

; 生成所有可能的皇后位置组合
(define (queens board-size)
  (define (queen-cols k)
    (if (= k 0)
        (cons-stream empty-board the-empty-stream)
        (stream-filter
         (lambda (positions) (safe? positions))
         (stream-flatmap
          (lambda (rest-queens)
            (stream-map (lambda (new-row)
                         (adjoin-position new-row rest-queens))
                       (stream-enumerate-interval 1 board-size)))
          (queen-cols (- k 1))))))
  
  (queen-cols board-size))

; 辅助函数：stream-flatmap
(define (stream-flatmap proc s)
  (flatten-stream (stream-map proc s)))

(define (flatten-stream stream)
  (if (stream-null? stream)
      the-empty-stream
      (stream-append (stream-car stream)
                    (flatten-stream (stream-cdr stream)))))

; 初始空棋盘
(define empty-board '())

; 打印解决方案的辅助函数
(define (display-queens solution)
  (define (print-row row size)
    (do ((i 1 (+ i 1)))
        ((> i size))
      (display (if (= i row) "Q " ". ")))
    (newline))
  
  (for-each (lambda (row)
              (print-row row 8)
              (newline))
            solution))

; 获取前N个解决方案
(define (get-n-solutions n)
  (define (iter stream count)
    (if (or (= count 0) (stream-null? stream))
        '()
        (cons (stream-car stream)
              (iter (stream-cdr stream) (- count 1)))))
  (iter (queens 8) n))

; 获取第一个解决方案
(define first-solution (stream-car (queens 8)))
(display-queens first-solution)

; 获取前5个解决方案
; (map display-queens (get-n-solutions 5))
```

### 有哪些缺点？
既然流这么好用，为什么不把这个特性封装到编程语言中？
1. 将所有操作都delay，就会导致，当进行迭代时，为了最后将结果计算出来，会产生很大的过程表，这是其中一个缺点。
2. 当处理一个对象时，由于流的懒加载特性，你不知道它的状态目前是什么样的。当系统复杂度越来越大时，无法控制当前对象的状态，会导致很多bug。

