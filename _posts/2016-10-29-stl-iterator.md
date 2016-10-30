---
layout: post
title: "STL 迭代器"
description: ""
category: stl
tags: [stl, iterator]
---
{% include JB/setup %}

## 概要

迭代器是一种行为类似指针的对象，而指针的各种行为中最常见也最重要的就是 dereference
和 member access，因此，迭代器最重要的编程工作就是对 operator* 和 operator-> 进行重载。

## 迭代器的设计

如果可以针对所有数据容器设计一个通用的迭代器自然是我们最好的选择，然而，这是不可能做到的。
考虑数组和双向链表：针对数组而言， 我们可以通过 `typedef T* iterator;` 定义迭代器；
而双向链表的迭代器则不可能有这么精简的结构。

由于容器的多样性，不同的容器往往需要不同的迭代算法。将迭代器的实现交给容器本身，
才是最合理的。

虽然迭代器的实现交由容器本身实现，但各容器不能随便定义接口迥异的迭代器。
为保证迭代器的访问一致性，我们需要为迭代器实现一组一致的行为。

所有迭代器必须定义以下内嵌型别：

``` c++
template <class _Category, class _Tp, class _Distance = ptrdiff_t,
          class _Pointer = _Tp*, class _Reference = _Tp&>
struct iterator {
  typedef _Category  iterator_category;
  typedef _Tp        value_type;
  typedef _Distance  difference_type;
  typedef _Pointer   pointer;
  typedef _Reference reference;
};
```

所有迭代器必须重载以下运算符：

``` c++
template<class _Tp, class _Ref, class _Ptr>
struct _Iterator {
  typedef _Iterator<_Tp,_Tp&,_Tp*>             iterator;
  typedef _Iterator<_Tp,const _Tp&,const _Tp*> const_iterator;
  typedef _Iterator<_Tp,_Ref,_Ptr>             _Self;

  reference operator*() const;
  pointer operator->() const;

  _Self& operator++();
  _Self operator++(int);
  _Self& operator--();
  _Self operator--(int);

  bool operator==(const _Iterator& __x) const;
  bool operator!=(const _Iterator& __x) const;
};
```

双向链表的迭代器实现示例：

``` c++
struct _List_iterator_base {
  typedef size_t                     size_type;
  typedef ptrdiff_t                  difference_type;
  typedef bidirectional_iterator_tag iterator_category;

  _List_node_base* _M_node;

  _List_iterator_base(_List_node_base* __x) : _M_node(__x) {}
  _List_iterator_base() {}

  void _M_incr() { _M_node = _M_node->_M_next; }
  void _M_decr() { _M_node = _M_node->_M_prev; }

  bool operator==(const _List_iterator_base& __x) const {
    return _M_node == __x._M_node;
  }
  bool operator!=(const _List_iterator_base& __x) const {
    return _M_node != __x._M_node;
  }
};

template<class _Tp, class _Ref, class _Ptr>
struct _List_iterator : public _List_iterator_base {
  typedef _List_iterator<_Tp,_Tp&,_Tp*>             iterator;
  typedef _List_iterator<_Tp,const _Tp&,const _Tp*> const_iterator;
  typedef _List_iterator<_Tp,_Ref,_Ptr>             _Self;

  typedef _Tp value_type;
  typedef _Ptr pointer;
  typedef _Ref reference;
  typedef _List_node<_Tp> _Node;

  _List_iterator(_Node* __x) : _List_iterator_base(__x) {}
  _List_iterator() {}
  _List_iterator(const iterator& __x) : _List_iterator_base(__x._M_node) {}

  reference operator*() const { return ((_Node*) _M_node)->_M_data; }

#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

  _Self& operator++() {
    this->_M_incr();
    return *this;
  }
  _Self operator++(int) {
    _Self __tmp = *this;
    this->_M_incr();
    return __tmp;
  }
  _Self& operator--() {
    this->_M_decr();
    return *this;
  }
  _Self operator--(int) {
    _Self __tmp = *this;
    this->_M_decr();
    return __tmp;
  }
};
```

## 型别萃取

迭代器型别萃取指的是：萃取迭代器所指向对象的型别。我们为迭代器定义了一组一致的内嵌型别，
直接通过域运算符获取不就可以了吗？

我们之前说过：迭代器是一种行为类似指针的对象。既然如此，那么原生指针算不算迭代器？
算！事实上，vector 的迭代器就是原生指针。

所以，在迭代器没有专门实现，而是纯粹指针的情况下，我们不能利用一致的接口来萃取型别。
于是就有了如下的“型别萃取机”：

``` c++
//  Marking input iterators.
struct input_iterator_tag { };

//  Marking output iterators.
struct output_iterator_tag { };

// Forward iterators support a superset of input iterator operations.
struct forward_iterator_tag : public input_iterator_tag { };

// Bidirectional iterators support a superset of forward iterator
// operations.
struct bidirectional_iterator_tag : public forward_iterator_tag { };

// Random-access iterators support a superset of bidirectional iterator
// operations.
struct random_access_iterator_tag : public bidirectional_iterator_tag { };

template <class _Iterator>
struct iterator_traits {
  typedef typename _Iterator::iterator_category iterator_category;
  typedef typename _Iterator::value_type        value_type;
  typedef typename _Iterator::difference_type   difference_type;
  typedef typename _Iterator::pointer           pointer;
  typedef typename _Iterator::reference         reference;
};

template <class _Tp>
struct iterator_traits<_Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef _Tp*                        pointer;
  typedef _Tp&                        reference;
};

template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef const _Tp*                  pointer;
  typedef const _Tp&                  reference;
};
```

## 迭代器的一些通用函数

`distance` 求两个迭代器之间的“距离”

``` c++
template <class _InputIterator>
inline typename iterator_traits<_InputIterator>::difference_type
__distance(_InputIterator __first, _InputIterator __last, input_iterator_tag)
{
  typename iterator_traits<_InputIterator>::difference_type __n = 0;
  while (__first != __last) {
    ++__first; ++__n;
  }
  return __n;
}

template <class _RandomAccessIterator>
inline typename iterator_traits<_RandomAccessIterator>::difference_type
__distance(_RandomAccessIterator __first, _RandomAccessIterator __last,
           random_access_iterator_tag) {
  __STL_REQUIRES(_RandomAccessIterator, _RandomAccessIterator);
  return __last - __first;
}

template <class _InputIterator>
inline typename iterator_traits<_InputIterator>::difference_type
distance(_InputIterator __first, _InputIterator __last) {
  typedef typename iterator_traits<_InputIterator>::iterator_category
    _Category;
  __STL_REQUIRES(_InputIterator, _InputIterator);
  return __distance(__first, __last, _Category());
}
```

`advance` 迭代器步进函数

``` c++
template <class _BidirectionalIterator, class _Distance>
inline void __advance(_BidirectionalIterator& __i, _Distance __n,
                      bidirectional_iterator_tag) {
  __STL_REQUIRES(_BidirectionalIterator, _BidirectionalIterator);
  if (__n >= 0)
    while (__n--) ++__i;
  else
    while (__n++) --__i;
}

template <class _RandomAccessIterator, class _Distance>
inline void __advance(_RandomAccessIterator& __i, _Distance __n,
                      random_access_iterator_tag) {
  __STL_REQUIRES(_RandomAccessIterator, _RandomAccessIterator);
  __i += __n;
}

template <class _InputIterator, class _Distance>
inline void advance(_InputIterator& __i, _Distance __n) {
  __STL_REQUIRES(_InputIterator, _InputIterator);
  __advance(__i, __n, iterator_category(__i));
}
```
