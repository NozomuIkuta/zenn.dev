---
title: "過激派が教える！　useEffectの正しい使い方"
emoji: "👹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

Reactの**useEffect**は、フックの中でも使い方が難しいものの一つです。そこで、この記事では筆者が考えるuseEffectの望ましい使い方を皆さんに伝授します。

## 基本原則

技術やその要素の使い方を考えるにあたって、筆者が好んでいるのは基本原則を置いてそれに基づいて判断することです。ということで、この記事ではまず筆者が考えるReactの基本原則を紹介します。

筆者がもっとも重要視する原則は、**ReactはUIライブラリである**ということです。つまり、ReactにはUIの管理をさせるべきであって、その他のことはReactの役目ではないということです。Reactが難しいと思う人がいる場合、何でもかんでもReactにやらせようとするから余計に難しくなっているのだと思います。

例えばアプリケーションのロジックの管理やそれに付随するステートの管理はReactの役目ではないので、Reactの外部で処理するべきです。そのためには[Recoil](https://recoiljs.org/)などのライブラリが有用です。

当然ながらuseEffectについてもこの原則が当てはまります。つまり、**useEffectはUIの管理という目的のために使うのであって、それ以外の使い方は良くない**ということです。

もう一つの基本原則は、**Reactはコンポーネントベースのライブラリである**ということです。Reactでは、コンポーネントを通じてUIを管理します。つまり、コンポーネント内に書かれるコードは、**useEffectも含めてすべてそのコンポーネントのロジックでなければなりません**。言い方を変えれば、そのコンポーネントに閉じないロジックを書くのは良くなくて、useEffectもその例外ではないということです。

Reactにおいて、コンポーネントはマウントされたりアンマウントされたりします。コンポーネントがアンマウントされたらコンポーネントは画面から消えるのですから、コンポーネントがUIに与えたあらゆる影響は消えるべきです。このことから、useEffectを使う際の原則の一つとして、**クリーンアップ関数の無いuseEffectは不適格**であるということが言えます。クリーンアップ関数が無いuseEffectは、コンポーネントがアンマウントされたときにそのコンポーネントの影響が元に戻されないので、コンポーネントベースの原則から明らかに外れています。

## ケーススタディ

以上で、伝えるべき原則は伝え終わりました。以下では、さまざまな具体例を通じて、useEffectの良い使い方・良くない使い方を見ていきましょう。

### イベントハンドラを登録する系

**許容度:** 😃 文句なし。望ましいuseEffectの使い方

例えば、次のコンポーネントは、マウスやタッチでページ内を移動すると色が変わります。これを実現するためには、コンポーネントがレンダリングする要素ではなく、ページ全体に対してイベントハンドラを登録する必要があります。

```tsx
const Component = () => {
  const [h, setH] = useState(0);

  useEffect(() => {
    const handler = () => {
      setH(h => (h + 1) % 360);
    };
    window.addEventListener("pointermove", handler);
    return () => {
      window.removeEventListener("pointermove", handler);
    };
  }, []);

  return (
    <div
      style={{
        backgroundColor: `hsl(${h}, 100%, 50%)`,
        width: "100px",
        height: "100px"
      }}
    />
  );
};
```

このように、Reactの他の機能では賄えないようなDOM操作をしたい場合にはuseEffectが必要です。ReactコンポーネントはあくまでUI（react-domであればDOM）を管理するためのものですから、useEffectを必要なDOM操作のために使うのは望ましい使い方です。

他にも、1秒間に60回など、高頻度で発生するイベントを処理する場合にもuseEffectを使うことがあります。Reactの仕組みに乗って再レンダリングを行うとパフォーマンス上の問題があるという場合には、エスケープハッチとしてuseEffectを通じてDOM操作を行うことができます。

### データを取得する系

**許容度:** 🙃 場合により許容できるケースもある。しかし、より良い代替手段がそのうち登場するので将来性は乏しい

例えば、次のコンポーネントは、GitHubのAPIを使ってユーザーの情報を取得し、表示します。

```tsx
const Component = () => {
  const [user, setUser] = useState<null | { login: string }>(null);

  useEffect(() => {
    const abortController = new AbortController();
    fetch("https://api.github.com/users/uhyo", {
      signal: abortController.signal
    })
      .then(res => res.json())
      .then(user => setUser(user))
      .catch(reportError);

    return () => {
      abortController.abort();
    };
  }, []);

  return <div>{user?.login}</div>;
};
```

このコンポーネントはマウントされると`fetch`によりAPIを呼び出し、結果が得られたらその内容を表示します。この実装ではuseEffectのクリーンアップ関数を使っているので、コンポーネントがアンマウントされたときにAPIの呼び出しがされます。これにより、コンポーネントがアンマウントされたら外部に与えた全ての影響が消えるべきであるという原則が守られています。クリーンアップを忘れていたら0点です。

ただし、データの取得をuseEffectで行うのがいつでも許容されるわけではありません。なぜなら、**ReactはUIライブラリである**という原則があるからです。UIに関連しないデータの取得はuseEffectの利用方法として望ましくありません。

ここで取得したデータが、アプリケーションのUI以外の部分（コアロジックなど）に影響を与える場合には、useEffectを使うのは適切ではありません。その場合はRecoilなど、Reactの外部にデータ取得を移すべきです。データ取得にuseEffectを使っていいのは、あくまでデータ取得の用途がそのコンポーネント内に限られるときだけです。コンポーネント内に書かれるロジックはそのコンポーネントのためのものであり、useEffectもその例外ではありません。

たまに、「useEffectは**副作用**を担当するフックであり[^note_side_effect]、Reactの本来のやり方から外れたロジックを書くためのエスケープハッチである」という主張を見かけますが、筆者はこれには賛同しません。useEffectもれっきとしたコンポーネントロジックの一部であり、React的な使い方で使うべきものです。Reactのデフォルトの機能（レンダリングを通じたDOM操作）で表現できないロジックを実装するためにuseEffectがあるので、ある種のエスケープハッチではあります。しかし、皆さんが想像するような“副作用”は、たとえuseEffectを使ったとしてもアンチパターンだと思っています。あくまで“主作用”のためにuseEffectを使いましょう。

[^note_side_effect]: 以前「副作用」という言葉を使ったら「関数型プログラミングには副作用と主作用のような区別はなく、単に作用があるだけである」というような指摘をもらったのですが、筆者は最近ReactはReactというパラダイムであり、関数型プログラミングに無理に当てはめることにはあまり意味がないと思うようになりましたので、今回はReact用語として副作用という言葉を使っています。useEffectがEffectと名乗っているからにはReact用語として「作用」であることは疑いようがなく、コンポーネントロジックの一部として実行されるものが（主）作用、そうではなくコンポーネント外などに影響を及ぼしてしまうものを副作用と分類しています。

余談ですが、そもそもコンポーネントからデータを取得する場合は、将来的には`use`が推奨される方法になりそうです。そのため、この用途でuseEffectを使うことは無くなるでしょう。

### トラッキングの例

**許容度:** 😡 だめ

useEffectを使ってトラッキングを行う例を見てみましょう。

```tsx
const Component = () => {
  const [searchQuery, setSearchQuery] = useState("");

  useEffect(() => {
    track("search", { searchQuery });
  }, [searchQuery]);

  return (
    <input
      value={searchQuery}
      onChange={e => setSearchQuery(e.target.value)}
    />
  );
};
```

このコンポーネントは、入力された文字列をトラッキングするためにuseEffectを使っています。この例では、`searchQuery`が変更されるたびにトラッキングが行われます。

この例は、useEffectを使ってトラッキングを行う例としては典型的なものです。useEffectを使ってトラッキングを行うのは**ReactはUIライブラリである**という原則に反しています。トラッキングはUIに関連しないデータの取得であり、useEffectを使ってトラッキングを行うのは適切ではありません。

特に目に付く問題としては、useEffectの返り値がありません。この例では、一度トラッキング用のデータを送信したらそれを取り消す手段が無いため、クリーンアップできないのです。このように、useEffectから更新系の処理をすることは問答無用でアンチパターンです。

また、上の例ではuseEffectの依存配列がロジックの一部となっていることも気に入りません。筆者の考えでは、useEffectの依存配列というのは、useMemoの依存配列と同じく、最適化のために使うべきです。useEffectに依存配列を渡さない場合はレンダリングのたびに実行とクリーンアップが行われる挙動となり、これがデフォルトと考えるべきです。そこまで頻繁にエフェクトの適用とクリーンアップを行う必要がない場合に、最適化として依存配列を渡すのです。

つまり極論、useEffectに渡された依存配列が最適化を超えた意味を持っている場合は、そのuseEffectは不適格です。上の例ではuseEffectの意味だけ見れば、`searchQuery`が変化するよりもっと頻繁に`track`が実行されてもおかしくないように読めます。今はたまたまReactのランタイムがちょうど我々が意図した頻度で実行してくれているだけなのです。

さらに、Reactから提供されているlintルールではuseEffect等の依存配列が正しくない場合（内部で使用されている値が依存配列にない場合など）はエラーとなります。これを無視するのも当然ご法度です。普通にバグのもとですし、依存配列を上述のように捉えていれば、依存配列のルールを逸脱する必要は皆無です。

実際、[以前の記事で解説したように](https://zenn.dev/uhyo/articles/react-18-alpha-essentials#strictmode-%E3%81%A7%E3%81%AE-useeffect-%E3%81%AE%E6%8C%99%E5%8B%95%E3%81%AE%E5%A4%89%E5%8C%96)、React 18からはStrictMode使用時にuseEffectのコールバックが複数回実行されることがあります。その挙動が適用された場合は上のuseEffectからも意図しない回数のイベントが発火することになるでしょう。

つまり、過激派なので言い切りますが、**値の変化に反応するためにuseEffectを使うのは良くない**のです。

上の例では、useEffectを使わずに次のように実装したほうが、コードの意図をより的確に表しているのでよいでしょう。

```tsx
const Component = () => {
  const [searchQuery, setSearchQuery] = useState("");

  const handleChange = 
    (e: React.ChangeEvent<HTMLInputElement>) => {
      setSearchQuery(e.target.value);
      track("search", { searchQuery: e.target.value });
    };

  return <input value={searchQuery} onChange={handleChange} />;
};
```

### ページが変わったらトラッキングするやつ

**許容度:** 😈 あかんで

上の例の亜種として、ページが変わったらトラッキングするというありがちな例を見てみましょう。

```tsx
const Component = () => {
  const router = useRouter();

  useEffect(() => {
    track("pageview", { page: router.pathname });
  }, [router.pathname]);

  return <div>...</div>;
};
```

SPAでは、ページの遷移をルーターと呼ばれる機構が担っています。上の例のuseRouterはルーターライブラリから提供されるフックという想定です。

この例も、前の節で述べた通りの問題を持っています。当然、useEffectのよい使い方ではありません。

先ほどと同じようにページ遷移を発生させる側のコードでトラッキングも済ませるのが一つの手ですが、その方法だと同じコードがアプリケーション中に分散してしまうのに加えてただ、高級なルーターライブラリだとページ遷移に割り込む機能などもあり、ページ遷移側で処理するのはそもそも難しそうです。望むらくは、React外のところでページ遷移を検知できる機構がルーターライブラリから提供されているべきです。ルーターに`onRouteChange`みたいなのが生えているイメージです。そのようなAPIであれば、ページ遷移とロジックレベルで紐づいた形でトラッキングを行うことができます。

つまり、たとえばReact Routerであれば`history.listen`を使ってトラッキングを行うのが正しい方法となります。Next.jsであれば`router.events`を経由して必要なイベントを登録することができます。次はNext.jsの例です。

```tsx:正しい方法（Next.jsの例）
const RouteTracker = () => {
  const router = useRouter();

  useEffect(() => {
    const handler = (url: string) => {
      track("pageview", { page: url });
    };
    router.events.on("routeChangeComplete", handler);

    return () => {
      router.events.off("routeChangeComplete", handler);
    };
    }
  }, [router]);

  return null;
};
```

この例ではまだuseEffectを使用していますが、「useEffectを使って`router.pathname`の変化に反応する」という問題のあるメンタルモデルを脱却し、routerに対してイベントを登録するという意味のコードになっています。きちんとクリーンアップされるエフェクトとなっており、元々の例の問題が解消されています。

ただ、このコンポーネントは常にnullを返していることからUIに寄与しているとは言い難く、100点満点のコンポーネントとは言えません。Next.jsがフックを介してしかルーターへのアクセスを提供してくれないため苦肉の策としてこうなっているという例です。

### useRefでエフェクトの実行回数を制御するやつ

**許容度:** 🥶 延命措置でしかない

React 18のStrictModeでuseEffectのコールバックが複数回実行されるという挙動に対して、useRefを使って回数を制限するというワークアラウンドがあります。次のようなコードです。

```tsx
const NicePage = () => {
  const eventFiredRef = useRef(false);
  useEffect(() => {
    // React 18のStrictModeで2回発火するので対策
    if (!eventFiredRef.current) {
      track("view", { page: "NicePage" });
      eventFiredRef.current = true;
    }
  }, []);

  return <div>...</div>;
};
```

このコンポーネントでは、useEffectを用いてページの表示をトラッキングしています。しかし、React 18のStrictModeではuseEffectのコールバックが2回実行されるため、対策としてuseRefを使って1回だけ実行するようにしています。

こうすると確かにReact 18のStrictMode下でも1回だけトラッキングされるようになりますが、お察しの通り、根本的な解決策ではありません。この場合の問題は、本来Reactの外で取り扱うべき問題をわざわざuseEffectを使ってReactを介して取り扱っていることにあります。無駄なレイヤーが挟まることで、このように無駄なワークアラウンドも必要になってしまうのです。

React 18が登場した際にStrictModeでuseEffect関連の対応に追われて、Reactは何でこんなにややこしいんだと思った方もいるかもしれません。しかし、大抵の場合はuseEffectを使った時点で間違ったやり方であり、道が整備されていないところを進んで歩きにくいと文句を言っているようなものです。

### useRef+useEffectの例としてよく出てくるタイマーのやつ

たとえば、次のような要件のコンポーネントを考えましょう。

- 1秒ごとにカウントアップするカウンターを表示する。
- チェックボックスがチェックされている場合はカウントアップを停止する。

#### 愚直な実装

この要件を愚直に実装するとこのようになります。

```tsx
const Timer = () => {
  const [count, setCount] = useState(0);
  const [paused, setPaused] = useState(false);

  useEffect(() => {
    if (paused) {
      return;
    }
    const timerId = setInterval(() => {
      setCount((c) => c + 1);
    }, 1000);
    return () => {
      clearInterval(timerId);
    };
  }, [paused]);

  return (
    <div>
      <div>{count}</div>
      <label>
        <input
          type="checkbox"
          checked={paused}
          onChange={(e) => setPaused(e.currentTarget.checked)}
        />
        pause
      </label>
    </div>
  );
};
```

**許容度:** 🥰 useEffectの使い方としては完璧

#### useIntervalによる実装

ただ、上の実装だと挙動に問題があります。それは、チェックボックスがチェックされた時点で1秒のカウントを開始するため、1秒未満の間隔でチェックボックスを連打するとカウントが全く進まないのです。

要件の定義次第ですが、1秒ごとに判定があり、そのタイミングでチェックボックスがチェックされていなければカウントアップするという要件だとしましょう。この場合、[Dan Abramov氏によるよく知られたuseIntervalというフック](https://usehooks-ts.com/react-hook/use-interval)を使えば次のように書くことができます。

```tsx
const Timer = () => {
  const [count, setCount] = useState(0);
  const [paused, setPaused] = useState(false);

  useInterval(() => {
    if (paused) {
      return;
    }
    setCount((c) => c + 1);
  }, 1000);

  return (
    /* ... */
  );
};
```

このフックではコンポーネントがマウントされている間常にタイマーが動き続けますが、コールバック関数からは常に最新のstate（`paused`）が参照できるというのが特徴です。これを実現するために、常に最新のコールバック関数をuseRefで作ったrefオブジェクトに入れ、useEffect内でsetIntervalで作られたタイマーからはrefオブジェクトに入っているコールバック関数を呼び出すようになっています。useRefを使わないと、useEffectでタイマーを作った時点のstateしか参照できなくなってしまいます。

```tsx:useIntervalの実装（上記の記事から引用）
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback)

  // Remember the latest callback if it changes.
  useIsomorphicLayoutEffect(() => {
    savedCallback.current = callback
  }, [callback])

  // Set up the interval.
  useEffect(() => {
    // Don't schedule if no delay is specified.
    // Note: 0 is a valid value for delay.
    if (!delay && delay !== 0) {
      return
    }

    const id = setInterval(() => savedCallback.current(), delay)

    return () => clearInterval(id)
  }, [delay])
}
```

このように、useEffect内外や複数のuseEffect間で値を受け渡す必要がある場合にuseRefを交えたロジックを書くことになります。言うなれば、useRefというのはエフェクト（やイベントハンドラなど）用の変数置き場と言えます。

このような実装に対する筆者の評価はというと、

**許容度:** 🤯 ちゃんと隠蔽しているなら……

この方法はeasyさを優先してかなりhackyなことをしているイメージです。熟達者以外が真似することはお勧めしません。そもそも、useEffect間で値を受け渡すという事態が尋常ではなく、原則の範囲内のやり方では起こりません。一応、useIntervalというフックにhackyな部分をうまく隠蔽できているのでそこを評価して、絶対にやるなという程ではない印象です。

#### useEffectの原則を守った実装

何とか筆者の原則を守って実装すると、たとえばこんな感じの実装が考えられます。

```tsx
const Timer = () => {
  const [initialTime] = useState(Date.now());
  const [count, setCount] = useState(0);
  const [paused, setPaused] = useState(false);

  useEffect(() => {
    if (paused) {
      return;
    }
    const now = Date.now();
    let nextTime = now + 1000 - ((now - initialTime) % 1000);
    const loop = () => {
      setCount((c) => c + 1);
      const now = Date.now();
      nextTime = now + 1000 - ((nextTime - now) % 1000);
      const diff = nextTime - now;

      timerId = setTimeout(loop, diff);
    };
    const diff = nextTime - now;
    let timerId = setTimeout(loop, diff);
    return () => {
      clearTimeout(timerId);
    };
  }, [paused, initialTime]);

  return (
    /* ... */
  );
};
```

**許容度:** 😎 満足感はある

この実装は、最初に起点となる時刻をステートに記憶しておくことで、useEffectからsetTimeoutを付けたり消したりしても常に1秒周期でタイマーが動くようになっています（タイマーの大雑把さはさておいて）。ややこしい実装になっていることは否定しませんが、一つのuseEffectに処理が閉じていてクリーンアップもちゃんとされるという、正しさの面では文句のない実装になっています。

ややこしいとは言っても、タイマーの処理というのは元々ややこしいものです。実際のアプリでは単純にタイマーを動かすだけでは不足で、タブがバックグラウンドに行った場合どういう処理にするのかなど考えることが多くあります（実は上の実装もその場合に少し配慮しています）。タイマーを扱うにはこのくらいのややこしさは覚悟しないといけないでしょう。

ということで、この例では、一見useRefなどを使わないとできなそうな実装も、工夫次第で原則を守ったまま実装できることを示しました。

:::details 余談

実は、細かな挙動に差異はあるものの、今回の要件であれば難しいことを考えずに次のようにすれば万事解決でした。

```tsx
const Timer2 = () => {
  const [{ count, paused }, setState] = useState({
    count: 0,
    paused: false
  });

  useEffect(() => {
    const timerId = setInterval(() => {
      setState((prev) =>
        prev.paused
          ? prev
          : {
              ...prev,
              count: prev.count + 1
            }
      );
    }, 1000);
    return () => {
      clearInterval(timerId);
    };
  }, []);

  return (
    <div>
      <div>{count}</div>
      <label>
        <input
          type="checkbox"
          checked={paused}
          onChange={(e) =>
            setState((prev) => ({
              ...prev,
              paused: !paused
            }))
          }
        />
        pause
      </label>
    </div>
  );
};
```

**許容度:** 💯 これまでの茶番は何だったのか

:::

## まとめ

この記事では、いくつかのuseEffectの使用例について、Reactの原則に基づいて評価しました。

結局、個々の例の評価についてはReact公式ドキュメントの[useEffect](https://react.dev/reference/react/useEffect)や[You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)といったページに書かれていることと大差ないものになりました。公式ドキュメントはすごいですね。今回の記事では、それらに対して少ない原則から説明を与えたところを評価してもらえればと思います。

ただ、この記事ではクリーンアップ関数のないuseEffectを全否定していますが、多分React公式はそこまで厳しい考え方ではなさそうです。この辺りが過激派要素です。

文中に出てきた原則やその帰結をまとめておきます。

**基本原則:**

- ReactはUIライブラリである。
- Reactはコンポーネントベースのライブラリである。

**帰結:**

- useEffectはUIの管理という目的のために使う。
- useEffectはコンポーネントロジックの一部である。
  - useEffectは“副作用”のためのものではない。
- クリーンアップ関数の無いuseEffectは不適格。
- useEffectの依存配列は最適化のためのものであり、最適化を超えた意味を持った依存配列は不適格である。
  - 値の変化に反応するためにuseEffectを使うのは良くない。
  - 依存配列のlintエラーを無視するのはご法度。

  帰結が多いように見えますが、本文を読んでいただくと分かる通り、少ない原則から導き出されるので別にややこしいものではありません。

  むしろ、useEffectを使うべきではない場面でuseEffectを使うほうが、あなたのReactアプリケーションをよほど複雑で理解しがたいものにするはずです。

  この記事を読んで楽しいuseEffectライフを送りましょう。
