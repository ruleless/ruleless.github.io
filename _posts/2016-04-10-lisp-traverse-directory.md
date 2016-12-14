---
layout: post
title: "lisp遍历目录 "
description: ""
category: lisp
tags: [语言, lisp]
---
{% include JB/setup %}

lisp是一门强大的语言无可否认，但有时候lisp还是免不了很二，比如：遍历目录！

不管是用C还是python想要遍历指定的目录都很简单，而且跨平台也很容易；
但对于lisp如果你想遍历一个目录下的所有文件，情况则没那么简单。
首先你得清楚路径名的概念，然后，对于不同的lisp实现你得编写不同的代码，
最后，你还得考虑windows与linux或者其他操作系统在目录结构上的差异。
总之，如果你一直很懒，一直都不曾编写通用的、跨平台的目录遍历函数，
那么每当你有遍历目录的需求时，你都会很纠结。

lisp下遍历目录的代码：

``` lisp
;; directory operation
(defun component-present-p (value)
  (and value (not (eql value :unspecific))))

(defun directory-pathname-p (p)
  (and
   (not (component-present-p (pathname-name p)))
   (not (component-present-p (pathname-type p)))
   p))

(defun pathname-as-directory (name)
  (let ((pathname (pathname name)))
    (when (wild-pathname-p pathname)
      (error "Can't reliably convert wild pathnames."))
    (if (not (directory-pathname-p name))
        (make-pathname
         :directory (append (or (pathname-directory pathname) (list :relative)) (list (file-namestring pathname)))
         :name nil
         :type nil
         :defaults pathname)
      pathname)))

(defun directory-wildcard (dirname)
  (make-pathname
   :name :wild
   :type #-clisp :wild #+clisp :nil
   :defaults (pathname-as-directory dirname)))

(defun list-directory (dirname)
  (when (wild-pathname-p dirname)
    (error "Can only list concret directory names."))
  (let ((wildcard (directory-wildcard dirname)))
    #+(or sbcl cmu lispworks)
    (directory wildcard)
    #+openmcl
    (directory wildcard :directories t)
    #+allegro
    (directory wildcard :directory-are-files nil)
    #+clisp
    (nconc
     (directory wildcard)
     (directory (clisp-subdirectories-wildcard wildcard)))
    #-(or sbcl cmu lispworks openmcl allegro clisp)
    (error "list-directory not implemented")))

(defun list-directory-tree (dirname)
  (let ((ret (cons "" nil)))
    (tool-list-directory-tree dirname ret)
    (setf ret (rest ret))))

(defun tool-list-directory-tree (dirname ret)
  (let ((filelist (list-directory dirname)))
    (dolist (file filelist)
      (If (not ret)
          (setf ret (cons (namestring file) nil))
          (setf (cdr (last ret)) (cons (namestring file) nil)))
      (when (directory-pathname-p file)
        (tool-list-directory-tree file ret)))))

#+clisp
(defun clisp-subdirectories-wildcard (wildcard)
  (make-pathname
   :directory (append (pathname-directory wildcard) (list :wild))
   :name nil
   :type nil
   :defaults wildcard))
```

在此基础上，我写了计算C和C++文件的源代码以及注释行行数的lisp函数。

``` lisp
(defun read-c++-source (source-path)
  (let ((file-list (cons "" nil)))
    (with-open-file (in source-path)
                    (do ((line (read-line in nil) (read-line in nil)))
                        ((not line))
                      (setf (cdr (last file-list)) (cons line nil))))
    (setf file-list (rest file-list))
    file-list))


;;count the lines of C++ source or C++ comment
(defun is-line-comment (line)
  (let ((new-line (remove #\Space line)))
    (if (= 0 (length new-line))
        nil
      (if (or (char= (elt new-line 0) #\*) (char= (elt new-line 0) #\/))
          t
        nil))))

(defun is-line-empty (line)
  (let ((new-line (remove-if #'(lambda (x) (or (char= x #\Space) (char= x #\Tab) (char= x #\Return) (char= x #\Newline))) line)))
    (if (= 0 (length new-line))
        t
      nil)))

(defun count-lines (source-path compare-func)
  (let ((ret 0) (file-list (read-c++-source source-path)))
    (dolist (line file-list)
      (when (funcall compare-func line)
        (incf ret)))
    ret))

(defun count-source-lines(source-path)
  (count-lines source-path #'(lambda (line) (and (not (is-line-empty line)) (not (is-line-comment line))))))

(defun count-comment-lines (source-path)
  (count-lines source-path #'(lambda (line) (is-line-comment line))))

(defun is-file-source-file (filename)
  (let ((reverse-filename (reverse filename)))
    (if (or (eql 0 (search "ppc." reverse-filename)) (eql 0 (search "c." reverse-filename)) (eql 0 (search "h." reverse-filename)))
        t
      nil)))

(defun count-source-lines-of-dir (dirname)
  (let ((filelist (list-directory-tree dirname)))
    (dolist (filename filelist)
      (when (is-file-source-file filename)
        (format t "~a    source-lines:~a comment lines:~a~%" filename (count-source-lines filename) (count-comment-lines filename))))))
```

注意：在上述列出目录树的函数中我以列表的形式将所有的目录及文件返回，当目录结构过大时，这个函数其实是无用的。
