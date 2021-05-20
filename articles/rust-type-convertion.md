---
title: "Rustの型変換"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - rust
published: true
---
Rustの型変換まわりを頭の整理がてらまとめました。

# cast
https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions

Rustでは`as`演算子でキャストを行えます。
`as`演算子は数値やbool, char, ポインタ等一部の型のみで利用できます。

`as`演算子はパニックを起こさないので安心して利用できます。

整数のキャストの例を載せておきます。
```rust
// 整数のサイズが変わらない場合はデータ自体に変更はなく、
// 型の解釈が変わるだけ
let x: u8 = 150;
let y = x as i8;
assert_eq!(y, -106);
assert_eq!(format!("{:b}", x), "10010110");
assert_eq!(format!("{:b}", y), "10010110");

// 整数のサイズが小さくなる場合は上位ビットが切り詰められる
let x: u16 = 500;
let y = x as i8;
assert_eq!(y, -12);
assert_eq!(format!("{:b}", x), "111110100");
assert_eq!(format!("{:b}", y), "11110100");

// 整数のサイズが大きくなる場合は符号ビットで埋める
let x1: u8 = 0;
let x2: i8 = -127;
let y1 = x1 as u16;
let y2 = x2 as i16;
assert_eq!(y1, 0);
assert_eq!(y2, -127);
assert_eq!(format!("{:b}", x1), "0");
assert_eq!(format!("{:b}", y1), "0");
assert_eq!(format!("{:b}", x2), "10000001");
assert_eq!(format!("{:b}", y2), "1111111110000001");
```

# Into/From
- [std::convert::Into](https://doc.rust-lang.org/std/convert/trait.Into.html)
- [std::convert::From](https://doc.rust-lang.org/std/convert/trait.From.html)

`Into`と`From`はある型から別の型への変換を行うtraitです。
`Into<T>`は`T`型へ変換できることを、`From<T>`は`T`型から変換できることを表しています。

`Into`と`From`はtraitを変換先に実装するのか、変換元に実装するのかという違いだけです。
呼び出し方が異なるだけでやっていることは全く同じになります。
そのため`Into`にはジェネリックな実装があり、`impl From<T> for X`を実装すると自動的に`X::from(t)`と`t.into::<X>()`が使えるようになります。

```rust
use std::convert::{From, Into};

#[derive(PartialEq, Eq, Debug)]
struct Color {
    alpha: u8,
    red: u8,
    green: u8,
    blue: u8,
}

impl From<u32> for Color {
    fn from(x: u32) -> Self {
        let alpha = (x >> 24) as u8;
        let red = (x >> 16) as u8;
        let green = (x >> 8) as u8;
        let blue = x as u8;

        Color {
            alpha,
            red,
            green,
            blue,
        }
    }
}

#[test]
fn test() {
    let c = Color {
        alpha: 0xff,
        red: 0x7f,
        green: 0x3f,
        blue: 0x0f,
    };

    let x: u32 = 0xff7f3f0f;

    assert_eq!(c, Color::from(x));

    // Fromしか実装していないがintoも使える
    assert_eq!(c, x.into());
}
```

# TryInto/TryFrom
- [std::convert::TryInto](https://doc.rust-lang.org/std/convert/trait.TryInto.html)
- [std::convert::TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html)

`TryInto`と`TryFrom`は`Into`と`From`に似ていますが、変換が失敗する可能性がある点が異なります。
（逆に言うと`Into`と`From`は変換に失敗するケースがあってはいけないということです。）

そのため`Result`を返すようになっています。
`TryFrom<T>`だけ実装すれば`TryInto<T>`が使えるようになる点は同じです。

```rust
use std::convert::{TryFrom, TryInto};

#[derive(PartialEq, Eq, Debug)]
struct PrimeNumber(u32);

impl TryFrom<u32> for PrimeNumber {
    type Error = &'static str;

    fn try_from(x: u32) -> Result<Self, Self::Error> {
        // めっちゃ雑な素数判定
        for n in 2..x {
            if x % n == 0 {
                return Err("not prime number");
            }
        }

        Ok(PrimeNumber(x))
    }
}

#[test]
fn test() {
    let x: u32 = 31;
    let y: u32 = 32;

    assert_eq!(Ok(PrimeNumber(31)), PrimeNumber::try_from(x));
    assert_eq!(Ok(PrimeNumber(31)), x.try_into());
    assert_eq!(
        Err("not prime number") as Result<PrimeNumber, _>,
        y.try_into()
    );
}
```

# FromStr/ToString
- [std::str::FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html)
- [std::string::ToString](https://doc.rust-lang.org/std/string/trait.ToString.html)

`FromStr`はその名の通り`&str`から変換できることを表すtraitで`From<str>`とほぼ同じです。
`ToString`は`FromStr`の逆で`String`への変換をするtraitですが、直接実装する必要はありません。

`ToString`は`Display`を実装する型へのジェネリックな実装を持っているため、`Display`を実装すると`ToString`も利用できるようになります。
`Display`のほうが汎用的なので、わざわざ`ToString`のみを実装する意味はないでしょう。

Rustの数値などの基本的な型は`FromStr`と`Display`を実装を実装しているので数値のパースに利用できます。

```rust
use std::fmt;
use std::str::FromStr;
use std::string::ToString;

#[derive(PartialEq, Eq, Debug)]
struct Color {
    red: u8,
    green: u8,
    blue: u8,
}

impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&format!(
            "#{:02x}{:02x}{:02x}",
            self.red, self.green, self.blue
        ))?;
        Ok(())
    }
}

impl FromStr for Color {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        fn parse_u8(c1: char, c2: char) -> Result<u8, String> {
            let mut s = String::with_capacity(2);
            s.push(c1);
            s.push(c2);

            u8::from_str_radix(&s, 16).map_err(|_| format!("cannot parse u8. {}", &s))
        }

        if s.len() != 7 {
            return Err("invalid length of str".to_string());
        }

        let chars: Vec<_> = s.chars().collect();

        if chars[0] != '#' {
            return Err("must be starts with #".to_string());
        }

        let red = parse_u8(chars[1], chars[2])?;
        let green = parse_u8(chars[3], chars[4])?;
        let blue = parse_u8(chars[5], chars[6])?;

        Ok(Color { red, green, blue })
    }
}

#[test]
fn test() {
    let c = Color {
        red: 0xff,
        green: 0x7f,
        blue: 0,
    };

    assert_eq!("#ff7f00", c.to_string());
    assert_eq!("#ff7f00", format!("{}", c));
    assert_eq!(Ok(c), Color::from_str(&"#ff7f00"));
    assert_eq!(true, Color::from_str(&"#hoge").is_err());
}
```

# IntoIter/FromIter
- [FromIterator](https://doc.rust-lang.org/std/iter/trait.FromIterator.html)
- [IntoIterator](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html)

`IntoIterator`はイテレータを生成できるようにするtraitです。
このtraitを実装することでfor式で回せるようになります。
`IntoIterator`はselfをmoveするため、そのまま実装すると1度しかイテレートできません。
何度でもイテレートできるようにするには参照に対してtraitを実装する必要があります。

`FromIterator`はイテレータから値を生成できるようにするtraitです。
このtraitを実装すると`collect`メソッドを使って値を生成できるようになります。
`FromIterator`を直接実装することは少ないかもしれませんが、`Vec`や`HashMap`といった型は`FromIterator`を実装しているおかげで`collect`できます。

```rust
use std::iter::{FromIterator, IntoIterator, Iterator};

#[derive(PartialEq, Eq, Debug)]
struct Bits(u32);

impl<'a> IntoIterator for &'a Bits {
    type Item = bool;
    type IntoIter = BitsIterator<'a>;

    fn into_iter(self) -> Self::IntoIter {
        BitsIterator {
            mask: 1,
            bits: self,
        }
    }
}

impl FromIterator<bool> for Bits {
    fn from_iter<T>(iter: T) -> Self
    where
        T: IntoIterator<Item = bool>,
    {
        let mut bits: u32 = 0;
        let mut mask: u32 = 1;

        for bit in iter.into_iter().take(32) {
            if bit {
                bits += mask;
            }

            mask = mask << 1;
        }

        Bits(bits)
    }
}

struct BitsIterator<'a> {
    mask: u32,
    bits: &'a Bits,
}

impl<'a> Iterator for BitsIterator<'a> {
    type Item = bool;

    fn next(&mut self) -> Option<Self::Item> {
        if self.mask == 0 {
            return None;
        }

        let x = (self.bits.0 & self.mask) != 0;
        self.mask = self.mask << 1;
        Some(x)
    }
}

#[test]
fn test() {
    let bits = Bits(10);
    let v: Vec<_> = bits.into_iter().take(4).collect();

    // 10 は 2進数で1010
    // 左からいてレートするのでこの並びになる
    assert_eq!(v, vec![false, true, false, true]);

    // IntoIteratorを実装すると、for式で直接回せるようになる
    for x in &bits {
        println!("{}", x);
    }

    // FromIteratorを実装するとcollectでイテレータから直接変換できるようになる
    assert_eq!(Bits(10), Bits::from_iter(vec![false, true, false, true]));
    assert_eq!(bits, vec![false, true, false, true].into_iter().collect());
}
```

# AsRef/AsMut
- [std::convert::AsRef](https://doc.rust-lang.org/std/convert/trait.AsRef.html)
- [std::convert::AsMut](https://doc.rust-lang.org/std/convert/trait.AsMut.html)

`AsRef`と`AsMut`は参照を別の型の参照に変換するためのtraitです。
`AsRef`がimmutable参照, `AsMut`がmutable参照です。

例えば`String`はUTF8でエンコーディングされていることが保証されている`u8`スライスとみなす事ができます。
`String`はもちろん, `str`, `Vec<u8>`, u8配列なども`AsRef<u8>`を実装しているので、trait boundsで利用することで汎用的にできます。
標準ライブラリでは`OsString`や`Path`といった`String`の類似型が相互に変換できるようになっているため、文字列で活用することが多くなるかと思います。

`AsMut`は変更可能という特性上、`AsRef`に比べ実装している型が限られています。
`str`は`AsMut<[u8]>`を実装していませんが、これは実装してしまうとUTF8として不正なバイト列を生成できてしまうからです。

```rust
use std::convert::{AsMut, AsRef};

fn hex_dump<T>(data: &T) -> String
where
    T: AsRef<[u8]>,
{
    let data = data.as_ref();
    let mut s = String::with_capacity(data.len() * 3);

    for x in data {
        s.push_str(&format!("{:02x} ", x));
    }

    s
}

fn zero_fill<T>(data: &mut T)
where
    T: AsMut<[u8]>,
{
    let data = data.as_mut();
    for x in data.iter_mut() {
        *x = 0;
    }
}

#[test]
fn test() {
    let mut ary = [1u8; 5];
    let mut vec: Vec<u8> = vec![1, 2, 10];
    let s = "abc".to_string();

    // AsRef
    assert_eq!(hex_dump(&ary), "01 01 01 01 01 ");
    assert_eq!(hex_dump(&vec), "01 02 0a ");
    assert_eq!(hex_dump(&s), "61 62 63 ");

    // AsMut
    zero_fill(&mut ary);
    zero_fill(&mut vec);
    // StringはAsMut<[u8]>を実装してない
    assert_eq!(hex_dump(&ary), "00 00 00 00 00 ");
    assert_eq!(hex_dump(&vec), "00 00 00 ");
}
```

# おわりに
Rustには他にも`Deref`や`Borrow`といった参照を変換するtraitが存在します。
このtraitはそれだけで1つの記事になりそうです。
そのうち整理してまとめようかと思います。
