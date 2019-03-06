---
title: '[Translation]Design is refactoring-The Lifetime of Food'
date: 2019-03-06 16:53:28
tags: rust
keyword: [rust, lifetime rule, borrowing rule]
categories: cpp
description: It is my first translation about rust programming(unfinished).
toc: true
---

## [Design is Refactoring-The Lifetime of Food](http://designisrefactoring.com/2016/08/10/the-lifetime-of-food/)
A Rusty Analogy using Millennial Stereotypes

Imagine you are out to dinner with some friends. You  are all stereotypical Millennials so photographing your meal is  expected. For every dish you order each of your friends borrows it,  takes a photograph, and then returns it to you.

Well, that’s the plan anyway. Some of your friends are not the most  reliable. Will they always return the right dish to you? Will they try  to sneak a bite? Human friends are so unreliable. But if your friends  were Rust functions you’d have nothing to worry about, thanks to Rust’s  rules around borrowing and lifetimes.
<!--more-->

> 想象一下，你准备和几个朋友一起去外面吃饭，你们都是典型的[千禧年一代](https://baike.baidu.com/item/%E5%8D%83%E7%A6%A7%E4%B8%80%E4%BB%A3/2683826?fr=aladdin)，所以拍照是意料之中的事情。对于你点的每一道菜，他们都会拿过去拍一张， 然后再还给你。
>
> 好了，这就是所谓的计划。他们之中有几个不是非常的靠得住，他们一定要把你点的还回来么?他们会吃掉几口么？人类的朋友不是非常可靠的。但是如果你的朋友是`Rust functions`,  那你就不用担心啦，这要感谢`Rust`围绕**借**的规则和**生命周期**规则。
>


Let’s model this dinner party in Rust and see if we can learn a thing or two about borrowing and lifetimes.

We’ll start off with the basics, a Dish that can be photographed.

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

Let’s introduce one of your friends. We’ll start off with your most  honest and trustworthy companion. Being honest this friend wants to  borrow a dish photograph it and return it. They don’t want you giving  them more than one dish – that could get confusing! When converted to a  Rust function, this friend looks like:

```rust
fn honest_friend(dish: &Dish) {
  dish.photograph("Honest Friend")
}
```

Our dinner-party now looks like:

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

```rust
fn honest_friend(dish: &Dish) {      //-- Borrow of the dish begins here
  dish.photograph("Honest Friend")   // |
}                                   //-- Borrow of the dish ends here
```

A new friend joins your table. This friend likes to play pranks, but  they don’t always go as expected. Your Trickster Friend says, “Loan me a  dish and I’ll give you a dish back. Trust me, I’m going to return to  you the same dish you loaned me.” You have your doubts.

```rust
fn trickster_friend(dish: &Dish) -> &Dish {
    dish.photograph("Incompenent Friend");

    // Tricks and Pranks ensue

    // Is this really the same dish you loaned?
    mystery_dish
}
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

Try both options and you’ll see that your trickster friend can not  prank you by giving you Pad Thai. The Pad Thai value is owned inside `trickster_friend` and is destroyed when `trickster_friend` ends. This is a bit easier to see if you inline the `trickster_friend` function into your `main` function

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

A third, very impatient friend arrives to our party. They don’t have  time for this “one plate at a time” nonsense; this friend wants you to  loan them two dishes at once:

```rust
fn impatient_friend(first_dish: &Dish, second_dish: &Dish) {
    first_dish.photograph("Impatient Friend");
    second_dish.photograph("Impatient Friend");
}
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

But your trickster friend sees this exchange and gets an idea. They  now know that they can’t return a plate of Pad Thai, but they figure  that if they borrow 2 plates and only return one, they can keep that  extra plate for themselves.

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

Remember what we first learned from our Trickster Friend? If a  function receives and returns a single reference then it must be the  same reference, otherwise you could return something that does not live  long enough. That’s easy for the Rust compiler to check as there’s only  one option. But this function receives two references and returns one.  How does the compiler ensure that the reference you return points to a  value that lives long enough?

In this particular code both our values – `doro` and `bruschetta` – live long enough. So it can be hard to see why the Rust compiler is confused. But what if we changed our code to:

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

If the Rust compiler ever has any questions about what reference a  function will return then that function must explicitly declare a  lifetime. Our trickster friend could declare one of four lifetimes:

Option One:

```rust
fn trickster_friend_option_one<'a>(first_dish: &'a Dish, second_dish: &Dish) -> &'a Dish
```

Option Two:

```rust
fn trickster_friend_option_two<'a>(first_dish: &Dish, second_dish: &'a Dish) -> &'a Dish
```

Option Three:

```rust
fn trickster_friend_option_three<'a>(first_dish: &'a Dish, second_dish: &'a Dish) -> &'a Dish
```

Option Four:

```rust
fn trickster_friend_option_four<'a, 'b>(first_dish: &'a Dish, second_dish: &'b Dish) -> &'a Dish
```

Let’s see the differences between those options:

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

Let’s try the `herring` code again:

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

Option Two is the inverse of Option One. The function must now return the `second_dish`. If we pass in `herring` as the first dish, our code still runs.

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

Is there more to lifetimes than this? Almost certainly. But I hope  that this analogy helps you grasp the fundamentals of why Rust has  lifetimes. If you want to learn more about fundamentals of Rust, maybe  read one of my earlier posts, [Rust via its Core Values](http://designisrefactoring.com/2016/04/01/rust-via-its-core-values/). I also suggest the [New Rustacean](http://www.newrustacean.com/) podcast, which has helped me understand quite a lot about Rust.