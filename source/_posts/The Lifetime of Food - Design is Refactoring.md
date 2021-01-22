---
title: '[Translation] The Lifetime of Food'
subtitle: 
date: 2019-03-06 16:53:28
tags: [rust, lifetime rule, borrowing rule]
keyword: [rust, lifetime rule, borrowing rule]
categories: translation
description: A Rusty Analogy using Millennial Stereotypes.
toc: true
---

## [Design is Refactoring-The Lifetime of Food](http://designisrefactoring.com/2016/08/10/the-lifetime-of-food/)
设计即重构 —— 食物的生命周期
A Rusty Analogy using Millennial Stereotypes
一个采用千禧年固化形象的 rust 式类比


Imagine you are out to dinner with some friends. You  are all stereotypical Millennials so photographing your meal is  expected. For every dish you order each of your friends borrows it,  takes a photograph, and then returns it to you.
> 想象一下，你准备和几个朋友一起去外面吃饭，你们都是典型的[千禧年一代](https://baike.baidu.com/item/%E5%8D%83%E7%A6%A7%E4%B8%80%E4%BB%A3/2683826?fr=aladdin)，所以拍照是意料之中的事情。对于你点的每一道菜，他们都会拿过去拍一张， 然后再还给你。
<!--more-->

Well, that’s the plan anyway. Some of your friends are not the most  reliable. Will they always return the right dish to you? Will they try  to sneak a bite? Human friends are so unreliable. But if your friends  were Rust functions you’d have nothing to worry about, thanks to Rust’s  rules around borrowing and lifetimes.
> 好了，这就是所谓的计划。他们之中有几个不是非常的靠得住。他们一定要把你点的还回来么？他们会吃掉几口么？人类朋友是非常不可靠的。但是如果你的朋友是 `Rust functions` ，  那你就不用担心啦，这要感谢 `Rust` 围绕 **借** 规则和 **生命周期** 规则。
>


Let’s model this dinner party in Rust and see if we can learn a thing or two about borrowing and lifetimes.
> 那就让我们使用 `Rust` 为这次晚餐聚会建模，然后看看我们能否学习一两点关于 **借用** 和 **生命周期** 的知识。

We’ll start off with the basics, a Dish that can be photographed.
> 我们将从基础开始，一盘用来拍照的菜。

```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    doro.photograph("Me");
    bruschetta.photograph("Me");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++doro.photograph%28%22Me%22%29%3B%0A++++bruschetta.photograph%28%22Me%22%29%3B%0A%7D)

We have a `Dish` struct which contains a name. There’s one function on `Dish`, `photograph`. You get your dishes from the hot new Italian/Ethiopian fusion restaurant and take their pictures.
> 我们有一个 `Dish` 结构体，里面包含一个 `name` 字段。`Dish` 还实现了一个函数 `photogragh`。 你在新开的火爆的意大利 / 埃塞俄比亚融合餐厅点了一些菜并拍下了它们的照片。

Let’s introduce one of your friends. We’ll start off with your most  honest and trustworthy companion. Being honest this friend wants to  borrow a dish photograph it and return it. They don’t want you giving  them more than one dish – that could get confusing! When converted to a  Rust function, this friend looks like:
> 让我们介绍一位你的朋友。我们将从最诚实最值得信任的伙伴开始。这位最信任的朋友想借一盘菜，拍照后再还给你。他们不让你给他们多于一盘菜，因为那将会很困扰。当将其写成 `Rust fucntion` ，这位朋友将会像这样：

```rust
fn honest_friend(dish: &Dish) {
  dish.photograph("Honest Friend")
}
```

Our dinner-party now looks like:
> 现在，我们的晚餐聚会是这样：

```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn honest_friend(dish: &Dish) {
    dish.photograph("Honest Friend");
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    honest_friend(&doro);
    honest_friend(&bruschetta);

    doro.photograph("Me");
    bruschetta.photograph("Me");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+honest_friend%28dish%3A+%26Dish%29+%7B%0A++++dish.photograph%28%22Honest+Friend%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++honest_friend%28%26doro%29%3B%0A++++honest_friend%28%26bruschetta%29%3B%0A%0A++++doro.photograph%28%22Me%22%29%3B%0A++++bruschetta.photograph%28%22Me%22%29%3B%0A%7D)

Note that your friend is not explicitly returning the Dish at the end  of their function. Thanks to Rust’s borrowing rules they don’t have to.  When your friend’s scope ends, the borrow ends.
> 要知道你的朋友并不是显式地在函数最后归还了那盘菜。由于 `Rust` 的 **借用** 规则，他们无需如此。一旦你朋友的 `scope` 结束，**借用** 也就结束了。

```rust
fn honest_friend(dish: &Dish) {      //-- Borrow of the dish begins here
  dish.photograph("Honest Friend")   // |
}                                   //-- Borrow of the dish ends here
```

A new friend joins your table. This friend likes to play pranks, but  they don’t always go as expected. Your Trickster Friend says, “Loan me a  dish and I’ll give you a dish back. Trust me, I’m going to return to  you the same dish you loaned me.” You have your doubts.
> 一个新朋友加入你们。这位朋友喜欢搞恶作剧，但也不是总是恶搞。你的骗子朋友说，“借我一盘菜，我会原样还给你的”。你有你的（如下）疑虑。

```rust
fn trickster_friend(dish: &Dish) -> &Dish {
    dish.photograph("Incompenent Friend");

    // Tricks and Pranks ensue

    // Is this really the same dish you loaned?
    mystery_dish
}
```
```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn trickster_friend(dish: &Dish) -> &Dish {
    dish.photograph("Trickster Friend");

    // Your friend is either returning the same dish or a different one.
    // Try running both options by uncommenting the assignments of mystery_dish

    // Option One: Returning a different one
    // let mystery_dish = &Dish { name: String::from("Pad Thai") };

    // Option Two: Returning the same one
    // let mystery_dish = dish;

    mystery_dish
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_dish = trickster_friend(&doro); 

    mystery_dish.photograph("Me");
    bruschetta.photograph("Me");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+trickster_friend%28dish%3A+%26Dish%29+-%3E+%26Dish+%7B%0A++++dish.photograph%28%22Trickster+Friend%22%29%3B%0A%0A++++%2F%2F+Your+friend+is+either+returning+the+same+dish+or+a+different+one.%0A++++%2F%2F+Try+running+both+options+by+uncommenting+the+assignments+of+mystery_dish%0A%0A++++%2F%2F+Option+One%3A+Returning+a+different+one%0A++++%2F%2F+let+mystery_dish+%3D+%26Dish+%7B+name%3A+String%3A%3Afrom%28%22Pad+Thai%22%29+%7D%3B%0A%0A++++%2F%2F+Option+Two%3A+Returning+the+same+one%0A++++%2F%2F+let+mystery_dish+%3D+dish%3B%0A%0A++++mystery_dish%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++let+mystery_dish+%3D+trickster_friend%28%26doro%29%3B+%0A%0A++++mystery_dish.photograph%28%22Me%22%29%3B%0A++++bruschetta.photograph%28%22Me%22%29%3B%0A%7D)

Try both options and you’ll see that your trickster friend can not  prank you by giving you Pad Thai. The Pad Thai value is owned inside `trickster_friend` and is destroyed when `trickster_friend` ends. This is a bit easier to see if you inline the `trickster_friend` function into your `main` function.
> 两种方案都试一下，你就会发现你的骗子朋友是不能给你一盘泰式（Pad Thai）来蒙骗过关。这个 Pad Thai 值由你的骗子朋友内部所有，并且在骗子朋友结束之后也随之销毁。如果将其内联到你的 `main` 函数中，就更加清晰了。

```rust
fn main() {
    let doro = Dish { name: String::from("Doro Wat") };

    let mystery_meal = {                                          //-- Mystery Meal is created here 
      let other_dish = &Dish { name: String::from("Pad Thai") };  // |  //-- Pad Thai is created here
      other_dish                                                  // |     |
    }                                                             // |  //-- Pad Thai is destroyed here
                                                                  // |
    me(mystery_meal)                                              //-- Mystery meal lives for longer than Pad Thai,
                                                                  //   so it can't contain a reference to Pad Thai. 
                                                                  //   Compiler Error
}
```

Our trickster friend shows us that if a function is going to receive and return a single borrowed reference it *has* to return the same reference it received. Even though our friend’s tricky plan failed we learned something from it!
> 我们这位骗子朋友向我们展示了：如果一个函数想要 **接收** 然后 **返回** 一个它自己拥有的 **借用引用** ，那就必须原样返回收到的 **引用**。尽管我们的骗子朋友没有得逞，我们也学到了一些新的东西。

A third, very impatient friend arrives to our party. They don’t have  time for this “one plate at a time” nonsense; this friend wants you to  loan them two dishes at once:
> 第三位加入到聚会的是一个尤为不耐心的朋友。他没有时间参与这种无意义的“一盘接着一盘”游戏；他想让你一次借给他两盘：

```rust
fn impatient_friend(first_dish: &Dish, second_dish: &Dish) {
    first_dish.photograph("Impatient Friend");
    second_dish.photograph("Impatient Friend");
}
```
```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn impatient_friend(first_dish: &Dish, second_dish: &Dish) {
    first_dish.photograph("Impatient Friend");
    second_dish.photograph("Impatient Friend");
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    impatient_friend(&doro, &bruschetta);

    doro.photograph("Me");
    bruschetta.photograph("Me");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+impatient_friend%28first_dish%3A+%26Dish%2C+second_dish%3A+%26Dish%29+%7B%0A++++first_dish.photograph%28%22Impatient+Friend%22%29%3B%0A++++second_dish.photograph%28%22Impatient+Friend%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++impatient_friend%28%26doro%2C+%26bruschetta%29%3B%0A%0A++++doro.photograph%28%22Me%22%29%3B%0A++++bruschetta.photograph%28%22Me%22%29%3B%0A%7D)

Just as with your honest friend, this works fine. We loan our two dishes to our impatient friend who takes their photos. When `impatient_friend` ends, so do the borrows.
> 就像你的那位值得信任的朋友那样，一切正常。我们借给不耐心朋友两盘菜，他拿去拍照。当这位朋友结束，借用也就结束了。

But your trickster friend sees this exchange and gets an idea. They  now know that they can’t return a plate of Pad Thai, but they figure  that if they borrow 2 plates and only return one, they can keep that  extra plate for themselves.
> 但是，你的捣蛋鬼朋友看到这种变化之后立马有了一个主意。他知道现在不能还一盘泰式，但是他想着如果借两盘，还一盘如何呢？

```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn trickster_friend_again(first_dish: &Dish, second_dish: &Dish) -> &Dish {
    first_dish.photograph("Trickster Friend");
    second_dish.photograph("Trickster Friend");

    first_dish
    //ha hah hah, I get to keep the second dish!
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = trickster_friend_again(&doro, &bruschetta);

    mystery_meal.photograph("Me!");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+trickster_friend_again%28first_dish%3A+%26Dish%2C+second_dish%3A+%26Dish%29+-%3E+%26Dish+%7B%0A++++first_dish.photograph%28%22Trickster+Friend%22%29%3B%0A++++second_dish.photograph%28%22Trickster+Friend%22%29%3B%0A%0A++++first_dish%0A++++%2F%2Fha+hah+hah%2C+I+get+to+keep+the+second+dish%21%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++let+mystery_meal+%3D+trickster_friend_again%28%26doro%2C+%26bruschetta%29%3B%0A%0A++++mystery_meal.photograph%28%22Me%21%22%29%3B%0A%7D)

Foiled again! If you run that you’ll get `error: missing lifetime specifier`.
> 再次受挫！如果你运行上述程序，你会得到 “error: missing lifetime specifier”。

Remember what we first learned from our Trickster Friend? If a  function receives and returns a single reference then it must be the  same reference, otherwise you could return something that does not live  long enough. That’s easy for the Rust compiler to check as there’s only  one option. But this function receives two references and returns one.  How does the compiler ensure that the reference you return points to a  value that lives long enough?
> 还记得第一次从这位捣蛋鬼朋友身上学习到什么了不？如果一个函数 **接收** 然后 **返回** 一个 **引用**，那他就必须是 **同样的引用**，否则你就会返回 **生命周期不够长** 的东西。由于只有这样唯一的一种选择，因此这对 Rust 编译器检查来说就很容易。但是对于这种接收了两个引用并且只返回其中一个的情况，编译器如何确保你返回的引用生命周期足够长呢？

In this particular code both our values – `doro` and `bruschetta` – live long enough. So it can be hard to see why the Rust compiler is confused. But what if we changed our code to:
> 在这种特定的代码中，两个值 —— `doro` 和 `bruschetta` —— 的生命周期都足够长。因此，对于 Rust 编译器而言，不会混淆。但是，如果我们如下修改我们的代码呢：

```rust
fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = {
      let herring = Dish { name: String::from("Pickled Herring") };   //-- Herring created here
      trickster_friend_again(&herring, &bruschetta)                   // |
    }                                                                 //-- Herring destroyed here


    mystery_meal.photograph("Me!");
}
```

If `trickster_friend_again` returns `bruschetta` then that code would be fine. But if it returns `herring` we’d try to photograph a dish that no longer existed. This is unsafe, so Rust won’t let us do it.
> 如果捣蛋鬼朋友返回 `bruschetta` ，那么代码会良好运行。但是如果返回 `herring` ，我们会对一盘不再存在的菜进行拍照。这是不安全的，所以 Rust 不允许我们如此操作。

If the Rust compiler ever has any questions about what reference a  function will return then that function must explicitly declare a  lifetime. Our trickster friend could declare one of four lifetimes:
> 如何 Rust 编译器对于函数将会返回什么引用有疑问，那么该函数就必须显式指明一个生命周期。我们的捣蛋鬼朋友可以（显式）指定一个生命周期，他有四种选择：

Option One:
> 选择一：

```rust
fn trickster_friend_option_one<'a>(first_dish: &'a Dish, second_dish: &Dish) -> &'a Dish
```

Option Two:
> 选择二：

```rust
fn trickster_friend_option_two<'a>(first_dish: &Dish, second_dish: &'a Dish) -> &'a Dish
```

Option Three:
> 选择三：

```rust
fn trickster_friend_option_three<'a>(first_dish: &'a Dish, second_dish: &'a Dish) -> &'a Dish
```

Option Four:
> 选择四：

```rust
fn trickster_friend_option_four<'a, 'b>(first_dish: &'a Dish, second_dish: &'b Dish) -> &'a Dish
```

Let’s see the differences between those options:
> 让我们看下这些不同选择的差异：

```rust
struct Dish {
    name: String,
}

impl Dish {
    fn photograph(&self, photographer: &str) {
        println!("Snap! {} photographed by {}", self.name, photographer)
    }
}

fn trickster_friend_lifetime_one<'a>(first_dish: &'a Dish, second_dish: &Dish) -> &'a Dish {
    first_dish.photograph("Trickster Friend");
    second_dish.photograph("Trickster Friend");

    //Uncomment below to try to return second_dish
    //second_dish

    //Comment this out when you try to return second_dish
    first_dish
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = trickster_friend_lifetime_one(&doro, &bruschetta);

    mystery_meal.photograph("Me!");
}
```

[Run](https://play.rust-lang.org/?code=struct+Dish+%7B%0A++++name%3A+String%2C%0A%7D%0A%0Aimpl+Dish+%7B%0A++++fn+photograph%28%26self%2C+photographer%3A+%26str%29+%7B%0A++++++++println%21%28%22Snap%21+%7B%7D+photographed+by+%7B%7D%22%2C+self.name%2C+photographer%29%0A++++%7D%0A%7D%0A%0Afn+trickster_friend_lifetime_one%3C%27a%3E%28first_dish%3A+%26%27a+Dish%2C+second_dish%3A+%26Dish%29+-%3E+%26%27a+Dish+%7B%0A++++first_dish.photograph%28%22Trickster+Friend%22%29%3B%0A++++second_dish.photograph%28%22Trickster+Friend%22%29%3B%0A%0A++++%2F%2FUncomment+below+to+try+to+return+second_dish%0A++++%2F%2Fsecond_dish%0A%0A++++%2F%2FComment+this+out+when+you+try+to+return+second_dish%0A++++first_dish%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+doro+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Doro+Wat%22%29+%7D%3B%0A++++let+bruschetta+%3D+Dish+%7B+name%3A+String%3A%3Afrom%28%22Bruschetta%22%29+%7D%3B%0A%0A++++let+mystery_meal+%3D+trickster_friend_lifetime_one%28%26doro%2C+%26bruschetta%29%3B%0A%0A++++mystery_meal.photograph%28%22Me%21%22%29%3B%0A%7D)

In this example we’ve declared one lifetime, given it to the reference passed to `first_dish` and our return value. Trickster friend now *must* return first_dish.
> 在这个例子中，我们指定了一个生命周期，将其给到传给 `first_dish` 的引用，还指定了一个返回值。现在，捣蛋鬼朋友必须返回 `first_dish` 。

Let’s try the `herring` code again:
> 让我们再试一下 `herring` 的代码：

```rust
fn trickster_friend_lifetime_one<'a>(first_dish: &'a Dish, second_dish: &Dish) -> &'a Dish {
    first_dish
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = {
      let herring = Dish { name: String::from("Pickled Herring") };   //-- Herring created here
      trickster_friend_lifetime_one(&doro, &herring)                  // |  this will always return doro
    }                                                                 //-- Herring destroyed here


    mystery_meal.photograph("Me!");
}
```

Using our current Trickster Friend this code will compile, because there is no way that `mystery_meal` will ever be `herring`.
> 现在，我们的捣蛋鬼朋友代码可以编译通过了，因为 `mystery_meal` 不可能会是 `herring`。

Option Two is the inverse of Option One. The function must now return the `second_dish`. If we pass in `herring` as the first dish, our code still runs.
> 选择一反过来就是选择二。该函数现在必须返回 `second_dish` 。如果我们传入  `herring` 作为第一参数，我们的代码仍然会运作。

```rust
fn trickster_friend_option_two<'a>(first_dish: &Dish, second_dish: &'a Dish) -> &'a Dish {
  second_dish
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = {
      let herring = Dish { name: String::from("Pickled Herring") };   //-- Herring created here
      trickster_friend_lifetime_two(&herring, &doro)                  // |   this will always return &doro
    }                                                                 //-- Herring destroyed here


    mystery_meal.photograph("Me!");
}
```

Option Three gives the same lifetime to both parameters so the  function could return either of them. Our herring code will fail to  compile regardless of parameter order because `mystery_meal` could be bound to a destroyed value.
> 选择三给予两个参数同样的生命周期，因此函数可以返回其中的任意一个。如论是什么传参顺序，我们的 `herring` 代码会编译失败。因为 `mystery_meal` 会绑定到一个已经销毁了的值。

```rust
fn trickster_friend_option_three<'a>(first_dish: &'a Dish, second_dish: &'a Dish) -> &'a Dish {
  // logic to determine if first_dish or second_dish is returned
  dish_to_return
}

fn main() {
    let doro = Dish { name: String::from("Doro Wat") };
    let bruschetta = Dish { name: String::from("Bruschetta") };

    let mystery_meal = {
      let herring = Dish { name: String::from("Pickled Herring") };   //-- Herring created here
      trickster_friend_lifetime_three(&herring, &doro)                // |   this could return doro or herring
    }                                                                 //-- Herring destroyed here


    mystery_meal.photograph("Me!");
}
```

In our dinner party Option Four is functionally the same as Option  One. We’ve declared two lifetimes but we’re never using the second one  making it irrelevant. I’m sure there are times when having two lifetimes  on a function is useful, but I haven’t come across them. [The book says you can have multiple lifetimes, but don’t explain why](https://doc.rust-lang.org/book/lifetimes.html#multiple-lifetimes). I’d love to know more about this.
> 在我们的晚餐聚会中，选择四功能上和选择一相同。我们指定了两个生命周期，但是并没有使用第二个，而是使其不相关。我能确定的是，指定两个生命周期的场景是有用的，但我自己并没有遇到过。[这本书介绍了可以指定多个生命周期的，但并没有解释原因](https://doc.rust-lang.org/book/lifetimes.html#multiple-lifetimes)，关于这点我想知道更多一些。

Is there more to lifetimes than this? Almost certainly. But I hope  that this analogy helps you grasp the fundamentals of why Rust has  lifetimes. If you want to learn more about fundamentals of Rust, maybe  read one of my earlier posts, [Rust via its Core Values](http://designisrefactoring.com/2016/04/01/rust-via-its-core-values/). I also suggest the [New Rustacean](http://www.newrustacean.com/) podcast, which has helped me understand quite a lot about Rust.

> 关于生命周期，有比这更多的资料么？几乎可以肯定。但是我希望这个类比可以帮助你抓住为何 Rust 有生命周期这个核心基础。如果你想学习更多关于 Rust 基础知识，或许可以阅读我的早期博文 —— [Rust via its Core Values](http://designisrefactoring.com/2016/04/01/rust-via-its-core-values/)。 我也同样建议[播客 New Rustacean](http://www.newrustacean.com/)，该播客帮助我理解了很多 Rust 知识。