---
title: 关于EditText几个小Tips
date: 2019-10-31 23:34:36
tags:
- Android
categories:
- Android
---

### 1.显示清除按钮

最简便的方式：利用 kotlin 拓展函数，通过 TextView#setCompoundDrawablesWithIntrinsicBounds 方法实现

<!--more-->

```kotlin
fun EditText.setupClearButtonWithAction() {

    addTextChangedListener(object : TextWatcher {
        override fun afterTextChanged(editable: Editable?) {
            val clearIcon = if (editable?.isNotEmpty() == true) R.drawable.ic_clear else 0
            setCompoundDrawablesWithIntrinsicBounds(0, 0, clearIcon, 0)
        }

        override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) = Unit
        override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) = Unit
    })

    setOnTouchListener(View.OnTouchListener { _, event ->
        if (event.action == MotionEvent.ACTION_UP) {
            if (event.rawX >= (this.right - this.compoundPaddingRight)) {
                this.setText("")
                return@OnTouchListener true
            }
        }
        return@OnTouchListener false
    })
}
```

### 2.禁止复制粘贴

```kotlin
fun TextView.disableCopyPaste() {
    isLongClickable = false
    setTextIsSelectable(false)
    customSelectionActionModeCallback = object : ActionMode.Callback {
        override fun onCreateActionMode(mode: ActionMode?, menu: Menu): Boolean {
            return false
        }

        override fun onPrepareActionMode(mode: ActionMode?, menu: Menu): Boolean {
            return false
        }

        override fun onActionItemClicked(mode: ActionMode?, item: MenuItem): Boolean {
            return false
        }

        override fun onDestroyActionMode(mode: ActionMode?) {}
    }
}
```

### 3.仅支持输入可打印的 ASCII 值

可打印的 ASCII 字符，可以通过遍历 1~128 ，将其强转为 char 类型实现

```java
 for (int i = 0; i < 128; i++) {
      System.out.print((char) i);
  }
```

运行结果

> !"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~

因为 " 、回车、 空格、'、 &、< 、> 需要进行转义，才能在 android 的 xml 中使用。对照如下

```
&#34; 或 &quot;   			"		
\n										回车
&#160;								空格
&#39; 或 &apos; 				'
&#38; 或 &amp;					&
&#60; 或 &lt;					<
&#62; 或 &gt;					>
```

若直接使用字符，比如 <，会报错：

> The value of attribute "android:digits" associated with an element type "EditText" must not contain the '<' character.

因此，在 xml 文件中给 EditText 指明 digits 属性为即可

```xml
android:digits="!#$%&amp;'()*+,-./0123456789:;&lt;=&gt;@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~&#34;"
```



[另一种方式](https://developer.android.com/reference/android/view/inputmethod/EditorInfo.html#IME_FLAG_FORCE_ASCII)，可以改变键盘的**初始状态**：imeOptions="flagForceAscii" 

也可以在代码中，通过 setTextFilter 来实现。



判断是否为 ASCII

```java
static boolean isASCII(String s)  {
    for (int i = 0; i < s.length(); i++) 
        if (s.charAt(i) > 127) 
            return false;
    return true;
}
```



