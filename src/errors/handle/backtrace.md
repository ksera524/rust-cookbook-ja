## 複雑なエラーシナリオのバックトレースを取得する

[![error-chain-badge]][error-chain] [![cat-rust-patterns-badge]][cat-rust-patterns]

このレシピでは複雑なエラーハンドリングとバックトレースの出力について紹介します。[`chain_err`]を使ってエラーを拡張し、新しいエラーを追加します。エラースタックは巻き戻すことができるため、エラーが発生した理由を理解のに良いでしょう。

下記のレシピでは`256`の`u8`へのデシリアライズを試します。SerdeからCSVにエラーが発生し、最終的にユーザーコードを表示します。

```rust
# extern crate csv;
#[macro_use]
extern crate error_chain;
# #[macro_use]
# extern crate serde_derive;
#
# use std::fmt;
#
# error_chain! {
#     foreign_links {
#         Reader(csv::Error);
#     }
# }

#[derive(Debug, Deserialize)]
struct Rgb {
    red: u8,
    blue: u8,
    green: u8,
}

impl Rgb {
    fn from_reader(csv_data: &[u8]) -> Result<Rgb> {
        let color: Rgb = csv::Reader::from_reader(csv_data)
            .deserialize()
            .nth(0)
            .ok_or("Cannot deserialize the first CSV record")?
            .chain_err(|| "Cannot deserialize RGB color")?;

        Ok(color)
    }
}

# impl fmt::UpperHex for Rgb {
#     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
#         let hexa = u32::from(self.red) << 16 | u32::from(self.blue) << 8 | u32::from(self.green);
#         write!(f, "{:X}", hexa)
#     }
# }
#
fn run() -> Result<()> {
    let csv = "red,blue,green
102,256,204";

    let rgb = Rgb::from_reader(csv.as_bytes()).chain_err(|| "Cannot read CSV data")?;
    println!("{:?} to hexadecimal #{:X}", rgb, rgb);

    Ok(())
}

fn main() {
    if let Err(ref errors) = run() {
        eprintln!("Error level - description");
        errors
            .iter()
            .enumerate()
            .for_each(|(index, error)| eprintln!("└> {} - {}", index, error));

        if let Some(backtrace) = errors.backtrace() {
            eprintln!("{:?}", backtrace);
        }
#
#         // In a real use case, errors should handled. For example:
#         // ::std::process::exit(1);
    }
}
```

表示されたエラーバックトレース:

```text
Error level - description
└> 0 - Cannot read CSV data
└> 1 - Cannot deserialize RGB color
└> 2 - CSV deserialize error: record 1 (line: 2, byte: 15): field 1: number too large to fit in target type
└> 3 - field 1: number too large to fit in target type
```

このエラーに関連した[`backtrace`]を表示するために`RUST_BACKTRACE=1`にしてレシピを実行してみましょう。

[`backtrace`]: https://docs.rs/error-chain/*/error_chain/trait.ChainedError.html#tymethod.backtrace
[`chain_err`]: https://docs.rs/error-chain/*/error_chain/index.html#chaining-errors
