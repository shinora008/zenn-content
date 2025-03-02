---
title: "React19にアップグレードに挑戦してみた話"
emoji: "❤️‍🔥"
author: "shinora008"
publication_name: "wed_engineering"
type: "tech"
topics: ["react", "javascript", "typescript", "nextjs"]
published: false
---

こんにちは、WED 株式会社でエンジニアをしている篠崎（[@sinora\_](https://x.com/sinora_)）です。

WED が開発・運営している、レシート買取アプリ「ONE」はアプリのみで展開していますが、
その「ONE」の運用で使用する管理画面は Next.js を採用しております。

最近管理画面の開発にアサインされることになり、フロントエンド側を書く機会がとても増えまして、自分の学習がてら 2024 年 12 月にリリースされた React19 のアップグレードにチャレンジしてみようと思い、
その軌跡を備忘録としてここに記述していこうと思います。

## なぜアップグレードしようと思ったのか？

forwardRef を使用する機会があり、React のドキュメントを読んでいるとこんな記述がありました。

> React 19 では、forwardRef は不要となりました。代わりに props として ref を渡すようにしてください。
> forwardRef は将来のリリースでは非推奨化される予定です。詳しくはこちらを参照してください。

[参照](https://ja.react.dev/reference/react/forwardRef)

これから forwardRef の実装をしようとしているのに、非推奨になると書かれると使わないようにしようって思いますよね？
現在使用している React のバージョンが 18 だったので、だったらこの機会にアップグレードしてみようかと思ったのがきっかけです。

また、forwardRef を使用して記述するとコードが若干書く量が増えて読みにくいので、それも解消できるという期待感もありました。

## 結論

React19 へのアップグレードは一旦見送りました。理由と致しましては、管理画面ということもありフォームを使っているコンポーネントが非常に多い為、動作確認等も時間がかかりそうで、しっかりとスケジュールを組んで取り組む必要があると判断しました。

私個人としては、初めてのアップグレードのチャレンジで使用しているライブラリや言語のアップグレードがいかに大変なのかを身をもって経験することができた為、React の理解を深めるいい機会になりました。

## やったこと

下記の順番でやっていきました。

1. React のアップグレード
2. Next.js のアップグレード
3. ビルドエラーの解消
4. E2E テストの実行

E2E テストを実行したところ、膨大なコード修正が必要だということがわかり今回はアップグレードを断念しました。ビルドができた時点で達成できたと思い込んでいたので、かなりの精神的ダメージがありました。

ただ、E2E テストのおかげでそれに気がつけたので、テストを作成してくれた先輩方にとても感謝しています(自分が書けてないので余計に)

アップグレードしていく中で印象に残った出来事を 3 つほど紹介させていただきます。

- forwardRef を削除してみた
- Next.js のアップグレード
- form の破壊的変更

## forwardRef を削除してみた

私がアップグレードをしようと思ったきっかけの一つである forwardRef の廃止。

forwardRef がどんな風に記述が変わったのか私のケースでご紹介していきます。

これまでは useRef を使用する場合は子コンポーネントに下記のような記述が必要で、ref を Props の値として渡すことができませんでした。

旧

```ts
type Props = {
  name: string;
};

const ChildComponent = React.forwardRef<HTMLDivElement, Props>(
  ({ name }: Props, ref) => {
    return (
      <>
        <div ref={ref}>{name}</div>
      </>
    );
  }
);

TextInput.displayName = "childComponent";
export default ChildComponent;
```

それが React19 ですと下記のように階層が一つ減らすことができます。

React19

```ts
type Props = {
  name: string;
  ref: React.Ref<HTMLDivElement>;
};

const ChildComponent = ({ name, ref }: Props) => {
  return (
    <>
      <div ref={ref}>{name}</div>
    </>
  );
};

export default ChildComponent;
```

初めて useRef を使用したときに、なぜ ref を Props として渡すことができないんだろう？と疑問だったので、私にとってはより自然に書くことができるようになった印象です。

## Next.js のアップグレード

こちらは自動でファイルを書き換えてくれるのが非常に印象的だったのでご紹介します。私は Next.js のバージョンを上げることも初めてだったので、変更されたファイルの多さにビビりました。アップグレード方法は公式に記載があるとおりで、下記のコマンドを実行しただけです。

```
npx @next/codemod@canary upgrade latest
```

[参照](https://nextjs.org/docs/app/building-your-application/upgrading/version-15)

変更内容は公式を参照にしていただければと思うのですが、例の一つを挙げると、props の値の受け取り方が変更された為従来の書き方だとエラーになるんですよね。

params、searchParams のいずれも Promise になりました。それにより型の変更と、await をする必要が出てきますが、上記のコマンドを打つだけで自動で書き換えてくれます。

![GitHubスクリーンショット](/images/20250303-react19-upgrade/github-screenshot.png)

それをまとめて 92 ファイルほどコードが自動で書き変わったので、ちゃんとビルドできるのか心配でしたが、問題なくビルドができ非常に簡単だったのでありがたかったです。

## form の破壊的変更

こちらが原因が判明するまで時間がかかってしまいました。(次からちゃんとドキュメント読みます)

ビルドができたので、試しにテストに通してみました。まぁするとめちゃくちゃ落ちまくるんですよね。当たり前だとは思いますが。

テストが落ちた原因の調査で特にハマったのが、form の submit 自体は問題なく動作するが、一度バリデーションに引っかかると、全ての input の値がリセットされるというものです。

私が原因を探るにあたり、form については React Hook Form、バリデーションには Zod を使用しているので、そちらの影響かと思い込んでいたというのもあります。

時間がかかりながらもようやくバグの原因と思われる変更を見つけることができました。

[参照](https://ja.react.dev/blog/2024/12/05/react-19#form-actions)

> <form> アクションが成功すると、React は非制御コンポーネントの場合にフォームを自動的にリセットします。手動で <form> をリセットする必要がある場合は、新しい React DOM の API である requestFormReset を呼び出すことができます。

ちゃんと自動的にリセットしますと記載がありました。

今回のアップグレードにより、form のアクションが変更になったようですね。
弊社の管理画面は複数のフォームアクションを実装する必要があり、formAction を使用していました。そのため、ボタンを押してエラーが出るとフォームの値がリセットされてしまいます。

ここの解決方法自体は、3 つほど見つけることができたのですが、全てのフォームを変更するとなると影響範囲が大きく、隙間時間できるものではないなと判断し今回はアップグレードを断念しました。

今回は私が試したことをご紹介していきます。
また私と同様の現象で記事にされていた [zaru さん](https://zenn.dev/moozaru/articles/171572f2707eac)の記事の内容を参考にさせていただいております。

### サンプルコード

色々試すのにボタンが二つ存在するシンプルなコンポーネントを用意しました。

![シンプルフォームイメージ](/images/20250303-react19-upgrade/sample-form-image.png)
下記のように React Hook Form , Zod を使用し、formAction でそれぞれのボタンの挙動を変えています。

```ts
"use client";
import TitledInput from "@/components/Form/TitledInput";
import Textarea from "@/components/Textarea";
import { z } from "zod";

const draftSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("DRAFT"),
});

const publishSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("SUBMITTED"),
});

const Form = () => {
  const submit = async (formData: FormData) => {
    console.log("保存の処理", formData);
  };

  // React Hook Form等のコードは省略

  const draftFormAction = async (formData: FormData) => {
    console.log("下書き保存");
  };

  const publishFromAction = async (formData: FormData) => {
    console.log("送信");
    // onSubmitはZodのschemaに従ってバリデーションを行い最終的にsubmit関数を呼び出します。
    onSubmit(formData);
  };

  return (
    <form id="sample-form">
      <div className="absolute top-12 right-8 flex gap-x-2">
        <button
          type="submit"
          className="normal-button"
          formAction={draftFormAction}
        >
          下書き保存
        </button>
        <button
          type="submit"
          className="primary-button"
          formAction={publishFromAction}
        >
          送信
        </button>
      </div>
      <div>
        <TitledInput title="商品名" isRequired>
          <Textarea
            className="w-[160px] h-[93px]"
            {...register("name", {
              required: true,
            })}
            placeholder={`商品名`}
            errors={errors.name?.message}
          />
        </TitledInput>
      </div>
    </form>
  );
};

export default Form;
```

### 1. onSubmit を使用する

formAction を使用すると、バリデーションに引っかかるとエラーが出て、フォームがリセットされてしまうのですが、form の onSubmit を使用すればバリデーションに引っかかった時にリセットされることは無くなるとのことで試しに実装してみました。

```ts
"use client";
import TitledInput from "@/components/Form/TitledInput";
import Textarea from "@/components/Textarea";
import { z } from "zod";
import { FormEvent } from "react";

const Form = () => {
  const submit = async (formData: FormData) => {
    console.log("保存の処理", formData);
  };

  // React Hook Form等のコードは省略
  const handleSubmit = async (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);
    await onSubmit(formData);
  };

  return (
    <form id="sample-form" onSubmit={handleSubmit}>
      <div className="absolute top-12 right-8 flex gap-x-2">
        <button type="submit" className="normal-button">
          下書き保存
        </button>
        <button type="submit" className="primary-button">
          送信
        </button>
      </div>
      <div>
        <TitledInput title="商品名" isRequired>
          <Textarea
            className="w-[160px] h-[93px]"
            {...register("name", {
              required: true,
            })}
            placeholder={`商品名`}
            errors={errors.name?.message}
          />
        </TitledInput>
      </div>
    </form>
  );
};

export default Form;
```

バリデーションが効いて、フォームの値もリセットされることはなくなりましたが、これだと複数のボタンを実装する際に、onSubmit 時にどのボタンが押されたのかわからなくなるため、別の解決方法を模索しました。

### 2. useState を使用する

こちらは input のコンポーネントを useState で管理するものです。

```ts
const [value, setValue] = useState<string | number | undefined>(defaultValue);
<input id={name} name={name} defaultValue={value ?? undefined} ref={ref} />;
```

フォームの値のリセットを防ぐことはできたのですが、すべての input となると現実的ではないかなと思い、こちらも却下しました。

### 3. useActionState を使用する

React19 からは form に非同期関数を渡すことがサポートされ、さらに useActionState というフックが新たに加わりました。

こちらを使えば、それぞれのボタンの formAction から useActionState を使用することで最終的に onSubmit を呼びます。

ただ、現状の実装ですと form の action ではなく、button の formAction を使用することになります。
button の formAction を経由して関数を呼び出すと、formData の値がリセットされてしまうようです。

formData の値がリセットされると useActionState の state に formData の値を入れて、無理やり更新するという実装になりました。

form の action で useActionState の action を呼ぶと、state に値を入れずともリセットはされなかったです。

下記のコードは form の action を useActionState の action で呼び出した例になります。ボタンが二つとも同じ動きにはなってはしまっていますが、こちらですと問題なくリセットはされませんでした。

useActionState を使う際は通常このような使い方するのがほとんどだと思います。

```ts
"use client";
import TitledInput from "@/components/Form/TitledInput";
import Textarea from "@/components/Textarea";
import { z } from "zod";
import { useActionState } from "react";

type DraftSchemaType = z.infer<typeof draftSchema>;
type PublishSchemaType = z.infer<typeof publishSchema>;

const draftSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("DRAFT"),
});

const publishSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("SUBMITTED"),
});

const initialValue: DraftSchemaType = {
  name: "",
  status: "DRAFT",
};

const Form = () => {
  const submit = async (formData: FormData) => {
    console.log("保存の処理", formData);
  };

  // React Hook Form等のコードは省略

  const formSubmit = async (
    state: DraftSchemaType | PublishSchemaType,
    formData: FormData
  ) => {
    await onSubmit(formData);
    return state;
  };

  const [state, action] = useActionState(formSubmit, initialValue);

  return (
    <form id="sample-form" action={action}>
      <div className="absolute top-12 right-8 flex gap-x-2">
        <button type="submit" className="normal-button">
          下書き保存
        </button>
        <button type="submit" className="primary-button">
          送信
        </button>
      </div>
      <div>
        <TitledInput title="商品名" isRequired>
          <Textarea
            className="w-[160px] h-[93px]"
            {...register("name", {
              required: true,
            })}
            defaultValue={state.name}
            placeholder={`商品名`}
            errors={errors.name?.message}
          />
        </TitledInput>
      </div>
      <input type="hidden" {...register("status")} value={state.status} />
    </form>
  );
};

export default Form;
```

そして、下記のコードはボタンごとに呼び出す関数を変更したものです。ボタンごとに formAction を変更しており、state に form の値を入れてから state を返すように変更しました。

```ts
"use client";
import TitledInput from "@/components/Form/TitledInput";
import Textarea from "@/components/Textarea";
import { z } from "zod";
import { useActionState } from "react";

type DraftSchemaType = z.infer<typeof draftSchema>;
type PublishSchemaType = z.infer<typeof publishSchema>;

const draftSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("DRAFT"),
});

const publishSchema = z.object({
  name: z.string().min(5, { message: "5文字以上入力してください" }),
  status: z.literal("SUBMITTED"),
});

const initialValue: DraftSchemaType = {
  name: "",
  status: "DRAFT",
};

const Form = () => {
  const submit = async (formData: FormData) => {
    console.log("保存の処理", formData);
  };

  // React Hook Form等のコードは省略

  const draftFormAction = async (formData: FormData) => {
    console.log("下書き保存");
    action(formData);
  };

  const publishFromAction = async (formData: FormData) => {
    console.log("送信");
    action(formData);
  };

  const formSubmit = async (
    state: DraftSchemaType | PublishSchemaType,
    formData: FormData
  ) => {
    state.name = formData.get("name") as string;
    state.status = formData.get("status") as "DRAFT" | "SUBMITTED";
    await onSubmit(formData);
    return state;
  };

  const [state, action] = useActionState(formSubmit, initialValue);

  return (
    <form id="sample-form">
      <div className="absolute top-12 right-8 flex gap-x-2">
        <button
          type="submit"
          className="normal-button"
          formAction={draftFormAction}
        >
          下書き保存
        </button>
        <button
          type="submit"
          className="primary-button"
          formAction={publishFromAction}
        >
          送信
        </button>
      </div>
      <div>
        <TitledInput title="商品名" isRequired>
          <Textarea
            className="w-[160px] h-[93px]"
            {...register("name", {
              required: true,
            })}
            defaultValue={state.name}
            placeholder={`商品名`}
            errors={errors.name?.message}
          />
        </TitledInput>
      </div>
    </form>
  );
};

export default Form;
```

少々強引なやり方な気もしますし、とても読みづらいコードになってしまったのでこちらも断念しました。
先に紹介した解決方法よりかは、修正箇所が少なくなるので、まだ可能性はありそうです。

## まとめ

フロントエンドの知識が浅い中で、初めてのアップグレードの挑戦でしたが大変勉強になりました。

今回は修正箇所が多くアップグレードは見送りましたが、アップグレードするとどうなるのだろうと自ら進んで挑戦したことは結果よかったです。
新しく追加された機能も実際に触ってみないことには理解は難しく、私の場合はドキュメント読んだだけよりかは実際にコードを書いて手を動かすことで使い所もしっかり理解ができました。

また、フォームのリセットをオプションにするといった話も出ているそうなので、アップグレードはもう少し様子見でも良さそうです。

引き続き他の変更も見ながら React の理解を深めていこうと思います。
