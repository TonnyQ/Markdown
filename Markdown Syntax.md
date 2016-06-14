Markdown语言的简要规则

1. 标题，总共六级标题
    # 一级标题
    ## 二级标题
    ### 三级标题

2. 列表，无序列表使用(\*,\-,\+)和有序列表(1.2.3.)

    无序列表
        * a.first
        * b.second
        * c.third

    有序列表
        1. 1
        2. 2
        3. 3

3. 引用
    >路漫漫其修远兮，吾将上下而求索。

4. 图片与链接

    * 插入链接：[Baidu](www.baidu.com)
    * 插入图片：![Mou icon](http://mouapp.com/Mou_128.png)

5. 粗体和斜体，删除线
    * 斜体: *这里是斜体*
    * 粗体: **这里是粗体**
    * 粗斜: ***粗斜***
    * 删除线: ~~shanchu~~

6. 表格

| Tables    | Are           | Cool   |
| ----------|---------------|--------|
| col 3 is  | right kdke    | $120   |
| col is 2  | left          | $240   |

7. 代码
    + 简单代码：`System.out.println("hello,world");`
    + 代码块:
    ```java
    /*这是一个简单的java类*/
    public class HelloJava{
        public static void Main(string args[])
        {
            System.out.println("Hello,Java!");
        }
    }
    ```
8. 分割线<http://www.baidu.com>
***

9. Mathjax 公式

    $$ x = \dfrac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

10. 流程图
    ```flow
     st=>start: Start
     e=>end: End
     op1=>operation: My Operation
     sub1=>subroutine: My Subroutine
     cond=>condition: Yes or No?
     io=>inputoutput: catch something...
     st->op1->cond
     cond(yes)->io->e
     cond(no)->sub1(right)->op1
     ```

11. 时序图
    ```sequence
     Alice->Bob: Hello Bob, how are you?
     Note right of Bob: Bob thinks
     Bob-->Alice: I am good thanks!
     ```
