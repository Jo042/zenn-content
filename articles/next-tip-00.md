---
title: "Next.jsにおけるexport/importの記述簡略化について"
emoji: "🕌"
type: "tech"
topics:
  - "next"
  - "react"
  - "typescript"
  - "個人開発"
  - "大学生"
published: true
published_at: "2025-06-21 08:04"
---

## はじめに
現在Next.jsで開発しているWebアプリにおいて、ディレクトリとコードの整理をする際に行ったことをここに残しておきます。Web開発もNext.jsも初心者なので優しい目でご覧ください。

### 内容
現在Next.jsで開発しているWebアプリにおいて、以下のようにインポートが長くなるの嫌だなーと思っていたところ、いい改善を見つけたので紹介します。

現在↓
```
import { useState, useEffect } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import AreaSelect from "./AreaSelect";
import GenreFilter from "./GenreFilter";
import PlaceType from "./PlaceType";
import SearchButton from "../ui/button/SearchButton";
import SearchResults from "../results/SearchResults";
import type { Spot } from '../../pages/api/spot';
import DetailButton from "../ui/button/DetailButton";
import ResetButton from "../ui/button/ResetButton";
import GenreModal from "../modal/GenreModal";
import AreaModal from "../modal/AreaModal";
import DetailModal from "../modal/DetailModal";
import { areaCategories } from "@/src/data/areaOptions";
import { genreCategories } from "@/src/data/genreOptions";
import { detailCategories } from "@/src/data/detailOptions";
```

改善後↓
```
import { useState, useEffect } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import type { Spot } from '../pages/api/spot';
import SearchResults from "./results/SearchResults";
import * as Selected from "@/src/features/filters/"
import * as Button from "@/src/features/ui/button"
import * as Modal from "@/src/features/modal/index"
import * as Options from "@/src/features/options"
```

### 方法
まずexportしている側を以下のようなindex.tsなどでまとめてexportします。
```
import AreaModal from "./AreaModal";
import DetailModal from "./DetailModal";
import GenreModal from "./GenreModal";

export { AreaModal, DetailModal, GenreModal}
```

そしてimport側では以下のようにまとめてimportし、使用します。
```
import * as Modal from "@/src/features/modal/index"

~~~

<Modal.AreaModal
        initial={area}
        selectedArea={selectedArea}
        setSelectedArea={setSelectedArea}
        options={Options.areaOptions}
        onClose={() => setShowAreaModal(false)}
        onSave={(newSelected) => setArea(newSelected)}
      />
```

### 参考
[Next.jsのディレクトリ構成のベストプラクティスを知っていますか？](https://www.youtube.com/watch?v=ekUQ043k2TQ)


## まとめ
今回は初めての技術記事でしたが、独学で学んでいるWeb開発のアウトプットとしてyoutubeで見かけた内容を記事にしてみました。他にもnoteなどで発信もしているのでよかったら覗いてみてください^^


https://note.com/majo_ao/n/n3a64f889ce1e

https://x.com/majo_0814
