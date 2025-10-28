+++
date = '2025-10-26T17:24:03+09:00'
draft = false
title = 'How Many Words Appear in 17,751 Japanese Literary Works?'
categories= ['Programming']
tags= ['Elixir', 'Mecab']
+++

### What Is Aozora Bunko?

---

[Aozora Bunko](https://www.aozora.gr.jp/) is an online library that aggregates public-domain literature—mostly works from the Meiji era through early Shōwa. As of 26 October 2025 it hosts 17,751 titles. You can download the catalog from the [aozorabunko GitHub repository](https://github.com/aozorabunko/aozorabunko).

### So How Many Japanese Words Do Those 17,751 Works Contain?

---

I was curious, and I also wanted to get more comfortable with Elixir, so I downloaded the collection and wrote scripts to count the words.

Here’s the answer up front: **114,010 distinct words.**

### How I Counted Them

---

First I cloned the [aozorabunko repository](https://github.com/aozorabunko/aozorabunko).

Beware: the repository has over 5,000 commits. Cloning the full history takes around 10 GB and plenty of time. A shallow clone is all we need, so use `git clone --depth=1 git@github.com:aozorabunko/aozorabunko.git`.

Almost every literary work lives in an `.html` file. There are many images, JS, and CSS assets, but we can ignore them.

Before jumping into code, I mapped out the plan:

1. List every HTML file.
2. Extract the text from each file. (I don’t strictly separate the main body from other page elements; the extra text is negligible.)
3. Tokenize the text and collect the words.
4. Filter garbage (mojibake, punctuation, etc.) and deduplicate.

**Time to write some Elixir.**

The `Path` module offers handy file-finding functions. `Path.wildcard/2`, which accepts glob patterns, is perfect for this task.

```elixir
html_files = Path.wildcard("/Users/xxxxx/aozorabunko/**/*.html")
```

To pull text out of HTML we need a parser. [Floki](https://github.com/philss/floki) is the most popular parser in the Elixir ecosystem. It has a concise, pleasant API.

Another wrinkle: character encoding. Elixir only handles UTF-8 strings. Many Aozora pages are encoded in `Shift_JIS`, so we must detect the encoding and convert it to UTF-8.

For detection I used [charset_detect](https://github.com/ryochin/charset_detect); for conversion, [iconv](https://github.com/processone/iconv).

```elixir
html_files
|> Enum.map(fn file ->
  data = file |> File.read!()

  content =
    case CharsetDetect.guess!(data) do
      "UTF-8" ->
        IO.puts("valid data")
        data

      "Shift_JIS" ->
        IO.puts("invalid data, file: #{file}")
        :iconv.convert("SHIFT_JIS", "UTF-8", data)

      unknown_encoding ->
        IO.puts("unknown encoding, file: #{file}")
        :iconv.convert(unknown_encoding, "UTF-8", data)
    end

  content
  |> Floki.parse_document!()
  |> Floki.find("body")
  |> Floki.text()
end)
```

For tokenization I relied on the open-source morphological analyzer [MeCab](https://taku910.github.io/mecab/), wrapped via [mecab-elixir](https://github.com/tex2e/mecab-elixir).

MeCab does far more than split text: it returns part-of-speech tags, readings, dictionary forms, and more. Here’s an example:

```elixir
iex> Mecab.parse("今日は晴れです")
[%{"conjugation" => "",
   "conjugation_form" => "",
   "lexical_form" => "今日",
   "part_of_speech" => "名詞",
   "part_of_speech_subcategory1" => "副詞可能",
   "part_of_speech_subcategory2" => "",
   "part_of_speech_subcategory3" => "",
   "pronunciation" => "キョー",
   "surface_form" => "今日",
   "yomi" => "キョウ"},
 %{"conjugation" => "",
   "conjugation_form" => "",
   "lexical_form" => "は",
   "part_of_speech" => "助詞",
   "part_of_speech_subcategory1" => "係助詞",
   "part_of_speech_subcategory2" => "",
   "part_of_speech_subcategory3" => "",
   "pronunciation" => "ワ",
   "surface_form" => "は",
   "yomi" => "ハ"},
 %{"conjugation" => "",
   "conjugation_form" => "",
   "lexical_form" => "晴れ",
   "part_of_speech" => "名詞",
   "part_of_speech_subcategory1" => "一般",
   "part_of_speech_subcategory2" => "",
   "part_of_speech_subcategory3" => "",
   "pronunciation" => "ハレ",
   "surface_form" => "晴れ",
   "yomi" => "ハレ"},
 %{"conjugation" => "基本形",
   "conjugation_form" => "特殊・デス",
   "lexical_form" => "です",
   "part_of_speech" => "助動詞",
   "part_of_speech_subcategory1" => "",
   "part_of_speech_subcategory2" => "",
   "part_of_speech_subcategory3" => "",
   "pronunciation" => "デス",
   "surface_form" => "です",
   "yomi" => "デス"},
 %{"conjugation" => "",
   "conjugation_form" => "",
   "lexical_form" => "",
   "part_of_speech" => "",
   "part_of_speech_subcategory1" => "",
   "part_of_speech_subcategory2" => "",
   "part_of_speech_subcategory3" => "",
   "pronunciation" => "",
   "surface_form" => "EOS",
   "yomi" => ""}]
```

The fields map to the following concepts:

| Field | Japanese | Description |
|-----------|--------|-------------|
| `surface_form` | 表層形 | The form that actually appears in the text |
| `part_of_speech` | 品詞 | Part of speech |
| `part_of_speech_subcategory1` | 品詞細分類1 | Part-of-speech detail 1 |
| `part_of_speech_subcategory2` | 品詞細分類2 | Part-of-speech detail 2 |
| `part_of_speech_subcategory3` | 品詞細分類3 | Part-of-speech detail 3 |
| `conjugation_form` | 活用形 | Conjugated form in context |
| `conjugation` | 活用型 | Conjugation pattern |
| `lexical_form` | 原形 | Dictionary form |
| `yomi` | 読み | Reading (kana) |
| `pronunciation` | 発音 | Pronunciation (actual sound) |

Here’s the parsing function:

```elixir
parse_article = fn article ->
  Mecab.parse(article)
  |> Enum.reject(fn %{
                      "lexical_form" => lexical_form,
                      "part_of_speech" => part_of_speech,
                      "surface_form" => surface_form
                    } ->
    # Filter out particles, symbols, EOS markers, etc.
    case {lexical_form, part_of_speech, surface_form} do
      {"*", _, _} -> true
      {_, _, "EOS"} -> true
      {_, "助詞", _} -> true
      {_, "記号", _} -> true
      _ -> false
    end
  end)
  |> Enum.map(fn %{"lexical_form" => lexical_form} -> lexical_form end)
  |> Enum.uniq()
end
```

Finally, aggregate the word lists, remove duplicates, and strip out noise:

```elixir
File.read!("./words.txt")
|> String.split(",")
|> Enum.reject(fn str ->
  cond do
    # Detect mojibake Latin characters
    String.match?(str, ~r/[\x{00C0}-\x{024F}]/u) ->
      true

    # Single full-width digit
    String.match?(str, ~r/^[\x{FF10}-\x{FF19}]$/u) ->
      true

    # Single full-width Latin letter
    String.match?(str, ~r/^[\x{FF21}-\x{FF3A}\x{FF41}-\x{FF5A}]$/u) ->
      true

    # Single half-width alphanumeric character
    String.match?(str, ~r/^[0-9A-Za-z]$/) ->
      true

    # Single kana character (half-width or full-width)
    String.match?(
      str,
      ~r/^[\x{3040}-\x{309F}\x{30A0}-\x{30FF}\x{31F0}-\x{31FF}\x{FF66}-\x{FF9D}]$/u
    ) ->
      true

    true ->
      false
  end
end)
|> tap(fn words ->
  File.write!("./clean_words.txt", Enum.join(words, ","))
end)
```

**Here are 100 sample words from the cleaned list:**

立ち寄る,いただく,ありがとう,ござる,ます,はじめて,おいで,なる,方,ため,おさめる,ある,本,ファイル,形式,読み方,紹介,する,基本,フォーマット,各,登録,作品,原則,種,用意,いる,それぞれ,特徴,読む,必要,道具,以下,とおり,です,テキスト,データ,できる,最も,シンプル,軽い,ルビ,ふりがな,入力,れる,もの,ない,圧縮,リンク,除く,解凍,復元,ソフト,入手,先,フリー,ウェア,シェア,以上,付属,その,最新,版,及び,こちら,ダウンロード,窓,杜,インターネット,標準,一部,社,リリース,リーダー,表示,いま,使い,縦,組み,製品,ページ,単位,構成,電子,ほとんど,つくる,上,専用,（株）,注意,マック,ユーザー,皆さん,改行,コード,多く,エディター,ワープロ,開く,行頭

Full source is available as a [Gist](https://gist.github.com/shuiRong/ec206728023d4e783d7c192e967119c7).
