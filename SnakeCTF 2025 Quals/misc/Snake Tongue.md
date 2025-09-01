# Snake Tongue
<img width="607" height="572" alt="Image" src="https://github.com/user-attachments/assets/93082daa-1d5e-41ae-8c24-57b085021a95" />

## Writeup

Upon connecting with ncat, we see a prompt suggesting the use of a **REPL** for the **Snake** language — an interactive execution environment:

<img width="923" height="147" alt="Image" src="https://github.com/user-attachments/assets/c5c728f7-73cf-4b74-a534-2db58490390c" />

The following **Lisp** code is provided:

```lisp
(require 'uiop)

(defmacro dhc (name args &body body)
  (if (fboundp name)
      (error "Can't do that, sorry")
      (labels ((spices (params body)
		 (if (null params)
		     `(progn ,@body)
		     `(lambda (,(car params))
			,(spices (cdr params) body)))))
	`(defun ,name (&rest args)
	   (reduce #'funcall args :initial-value ,(spices args body))))))

(defun please (x &optional env)
  (cond
    ((null x) nil)
    ((symbolp x)
     (gar x env))
    ((atom x) x)
    ((case (first x)
       (8 (second x))
       (1 (lastl (mapcar #'(lambda (y) (please y env)) (rest x))))
       (2 (sar! (second x) (please (third x) env) env))
       (<> (if (please (second x) env)
	       (please (third x) env)
	       (please (fourth x) env)))
       (? (let ((parms (second x))
		(code (cofree-comonad-absolutely '1 (rest2 x))))
	    (lambda (&rest args)
	      (please code (letsgo parms args env)))))
       (! (let ((name (second x))
		(args (list (first (third x))))
		(body (cdddr x)))
	    (eval `(dhc ,name ,args ,@body))))
       (t
	(apply (please (first x) env)
	       (mapcar #'(lambda (v) (please v env)) (rest x))))))))

(defun rest2 (x) (rest (rest x)))

(defun sar! (var val env)
  (if (assoc var env)
      (setf val (second (assoc var env)))
      (sgar! var val))
  val)

(defun gar (var env)
  (if (assoc var env)
      (second (assoc var env))
      (ggar var)))

(defun sgar! (var val)
  (setf (get var 'global-val) val))

(defun ggar (var)
  (let* ((default "lol rip")
	 (val (get var 'global-val default)))
    (if (eq val default)
	(error "WTF is ~a" var)
	val)))

(defun letsgo (vars vals env)
  (nconc (mapcar #'list vars vals) env))

(defparameter *dealwithit*
  '(format))

(defun init-please ()
  ;; Define the procedures as CL functions
  (mapc #'cope *dealwithit*))


(defun cope (f)
  (if (listp f)
      (if (functionp (second f))
	 (sgar! (first f) (symbol-function (second f))) 
	 (sgar! (first f) (second f)))
      ;; Otherwise, return the function directly
      (sgar! f (symbol-function f))))

(defun cofree-comonad-absolutely (op exps &optional if-nil)
  (cond ((null exps) if-nil)
	((= (length exps) 1) (first exps))
	(t (cons op exps))))

(defun lastl (list)
  (first (last list)))

(defun repl ()
  (init-please)
  (format t "Snake lang REPL, enjoy your stay.")
  (finish-output)
  (loop while t
	do (format t "~&>>> ")
	   (finish-output)
	   (princ (please (read) nil))))


(defun main ()
  (set-dispatch-macro-character #\# #\. #'(lambda (s x y) (declare (ignore s x y)) nil))
  (defparameter *flag* (let ((flag (uiop:getenv "FLAG")))
                 (if flag
                 flag
                 "REDACTED")))
  (repl))
```

### Analysis of special operators

In the `please` function, the first elements of the list are used as **opcodes** (**symbolic or numeric values ​​used to represent commands/instructions** in the **Snake** language:

```lisp
((case (first x)
  (8 (second x))                          ; Quote-like: (8 x) ⇒ x
  (2 (sar! (second x) (please (third x))  ; Assign: (2 var val)
  ...
```

So:

| Snake code          | Common Lisp Correspondent         |
| ------------------- | --------------------------------- |
| `(8 foo)`           | `quote` → returns `foo` as symbol |
| `(2 var val)`       | assigns `val` to symbol `var`     |
| `(get-val arg)`     | calls `get-val` with `arg`        |
| `(! name args ...)` | defines function dynamically      |

### The vulnerability

The function `(2 var val)` compiles to:

```
(sar! var (please val) env)
```

And in `sar!`, if `var` is not yet present in the local environment (`env`), then:

```
(sgar! var val) → (setf (get var 'global-val) val)
```

> If you assign a **symbol** to something, and it doesn't exist in the **local context**, it gets stored in the **global dictionary** — using `get/put`

And in the `cope` function:

```
(sgar! f (symbol-function f))
```

> I can assign a **real function** (like `symbol-value`) to a **Snake symbol**

We exploited it usign:

```
(2 get-val (8 symbol-value))
```

> This binds `get-val` to Common Lisp's `symbol-value` function.

Then, by calling:

```
(get-val (8 *flag*))
```

We're effectively doing:

```
(symbol-value '*flag*)
```

<img width="825" height="220" alt="Image" src="https://github.com/user-attachments/assets/d08332fa-5430-447a-a614-6139c5d260a7" />

Flag: **snakeCTF{pr0duct10n_re4dy_l4nguAge_ca33bd57dce38400}**
