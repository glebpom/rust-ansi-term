# rust-ansi-term [![](http://meritbadge.herokuapp.com/ansi-term)](https://crates.io/crates/ansi-term) [![Build Status](https://travis-ci.org/ogham/rust-ansi-term.svg?branch=master)](https://travis-ci.org/ogham/rust-ansi-term) [![Coverage Status](https://coveralls.io/repos/ogham/rust-ansi-term/badge.svg?branch=master&service=github)](https://coveralls.io/github/ogham/rust-ansi-term?branch=master)

This is a library for controlling colours and formatting, such as red
bold text or blue underlined text, on ANSI terminals.

### [View the Rustdoc](http://ogham.rustdocs.org/ansi_term/)


## Installation

This crate uses [Cargo](http://crates.io/), Rust's package manager.
You can use this crate by adding `ansi_term` to your Cargo dependencies:

```toml
[dependencies]
ansi_term = "*"
```


## Basic usage

There are two main data structures in this crate that you need to be concerned with: `ANSIString` and `Style`.
A `Style` holds stylistic information: colours, whether the text should be bold, or blinking, or whatever.
There are also `Colour` variants that represent simple foreground colour styles.
An `ANSIString` is a string paired with a `Style`.

(Yes, it’s British English, but you won’t have to write “colour” very often. `Style` is used the majority of the time.)

To format a string, call the `paint` method on a `Style` or a `Colour`, passing in the string you want to format as the argument.
For example, here’s how to get some red text:

    use ansi_term::Colour::Red;
    println!("This is in red: {}", Red.paint("a red string"));

It’s important to note that the `paint` method does *not* actually return a string with the ANSI control characters surrounding it.
Instead, it returns an `ANSIString` value that has a `Display` implementation that, when formatted, returns the characters.
This allows strings to be printed with a minimum of `String` allocations being performed behind the scenes.

If you *do* want to get at the escape codes, then you can convert the `ANSIString` to a string as you would any other `Display` value:

    use ansi_term::Colour::Red;
    use std::string::ToString;
    let red_string = Red.paint("a red string").to_string();


## Bold, underline, background, and other styles

For anything more complex than plain foreground colour changes, you need to construct `Style` objects themselves, rather than beginning with a `Colour`.
You can do this by chaining methods based on a new `Style`, created with `Style::new()`.
Each method creates a new style that has that specific property set.
For example:

    use ansi_term::Style;
    println!("How about some {} and {}?",
             Style::new().bold().paint("bold"),
             Style::new().underline().paint("underline"));

For brevity, these methods have also been implemented for `Colour` values, so you can give your styles a foreground colour without having to begin with an empty `Style` value:

    use ansi_term::Colour::{Blue, Yellow};
    println!("Demonstrating {} and {}!",
             Blue.bold().paint("blue bold"),
             Yellow.underline().paint("yellow underline"));
    println!("Yellow on blue: {}", Yellow.on(Blue).paint("wow!"));

The complete list of styles you can use are:
`bold`, `dimmed`, `italic`, `underline`, `blink`, `reverse`, `hidden`, and `on` for background colours.

Finally, you can turn a `Colour` into a `Style` with the `normal` method.
This will produce the exact same `ANSIString` as if you just used the `paint` method on the `Colour` directly, but it’s useful in certain cases: for example, you may have a method that returns `Styles`, and need to represent both the “red bold” and “red, but not bold” styles with values of the same type. The `Style` struct also has a `Default` implementation if you want to have a style with *nothing* set.

    use ansi_term::Style;
    use ansi_term::Colour::Red;
    Red.normal().paint("yet another red string");
    Style::default().paint("a completely regular string");


## Extended colours

You can access the extended range of 256 colours by using the `Fixed` colour variant, which takes an argument of the colour number to use.
This can be included wherever you would use a `Colour`:

    use ansi_term::Colour::Fixed;
    Fixed(134).paint("A sort of light purple");
    Fixed(221).on(Fixed(124)).paint("Mustard in the ketchup");

The first sixteen of these values are the same as the normal and bold standard colour variants.
There’s nothing stopping you from using these as `Fixed` colours instead, but there’s nothing to be gained by doing so either.


## Combining successive coloured strings

The benefit of writing ANSI escape codes to the terminal is that they *stack*: you do not need to end every coloured string with a reset code if the text that follows it is of a similar style.
For example, if you want to have some blue text followed by some blue bold text, it’s possible to send the ANSI code for blue, followed by the ANSI code for bold, and finishing with a reset code without having to have an extra one between the two strings.

This crate can optimise the ANSI codes that get printed in situations like this, making life easier for your terminal renderer.
The `ANSIStrings` struct takes a slice of several `ANSIString` values, and will iterate over each of them, printing only the codes for the styles that need to be updated as part of its formatting routine.

The following code snippet uses this to enclose a binary number displayed in red bold text inside some red, but not bold, brackets:

    use ansi_term::Colour::Red;
    use ansi_term::{ANSIString, ANSIStrings};
    let some_value = format!("{:b}", 42);
    let strings: &[ANSIString<'static>] = &[
        Red.paint("["),
        Red.bold().paint(some_value),
        Red.paint("]"),
    ];
    println!("Value: {}", ANSIStrings(strings));

There are several things to note here.
Firstly, the `paint` method can take *either* an owned `String` or a borrowed `&str`.
Internally, an `ANSIString` holds a copy-on-write (`Cow`) string value to deal with both owned and borrowed strings at the same time.
This is used here to display a `String`, the result of the `format!` call, using the same mechanism as some statically-available `&str` slices.
Secondly, that the `ANSIStrings` value works in the same way as its singular counterpart, with a `Display` implementation that only performs the formatting when required.