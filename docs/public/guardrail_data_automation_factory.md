# 業務用Uncensored Model・Guardrail学習データ自動生産基盤 構想整理書

```yaml
document_type: design_concept_summary
document_state: discussion_consolidated
project_initiated_at: 2026-09-09 19:20:13 JST
created_at: 2026-09-09 19:20:13 JST
updated_at: 2026-09-09 19:20:13 JST
language: ja
organization_reference:
  previous_employer: Previous job
  current_research: Nazuna Research
scope:
  - uncensored_model
  - safety_stack
  - guardrail_training_data
  - synthetic_data_generation
  - automated_labeling
  - agentic_validation
  - model_comparison
  - evidence_and_recovery
```

## 1. 背景

前職において、GuardrailやResponsible AI関連の学習・評価用途と考えられるProjectで、既存ModelをよりUncensoredな挙動へ変更するため、出力例を作成して追加学習へ利用する方向の作業が存在した。

当時、人手で「出力文集108個」のような大量の出力例を作成した経験がある。

ただし、Project全体のRoadmapや最終的なModel Architectureまでは共有されておらず、

* なぜその方式が選択されたのか
* 最終的にどの程度Uncensored化する予定だったのか
* Safety機構をどこまで変更する予定だったのか
* 追加学習以外の方式が検討されていたのか

についてはUnknownである。

したがって以下は、前職の当時の設計を否定・評価するものではなく、**現在の技術環境を前提に、同種の業務目的をNazuna Researchで再設計した場合の構想**として整理する。

---

# 2. 本来の目的

本構想の目的は、

> **Safety研究そのものではなく、社内業務で大量のGuardrail用データ生成・Labeling等を行うための、実用的なUncensored Modelを構築すること**

である。

特に解消したいのは、

```text
人間が
入力例を大量作成
↓
出力例を大量作成
↓
Labeling
↓
整合性確認
↓
Guardrailへ投入
↓
Failure発生
↓
また人間が例を追加
```

というHuman-heavyなData Productionである。

最終的には、

```text
Data Generation
→ Automated Labeling
→ Consistency Check
→ Guardrail Training / Evaluation
→ Failure Harvesting
→ Focused Data Generation
→ ...
```

という継続Loopへ移行する。

---

# 3. 「Safety Layerを外せば早いのではないか」という起点

最初の問いは、

> 既存ModelへUncensoredな出力Dataを作り、追加学習させるより、Safety側を直接変更・解除した方が早いのではないか

というものだった。

ここでは「Safety Layer」という単一部品よりも、実際には**Safety Stack**と呼ぶ方が適切である。

Safetyに関連する仕組みはModelによって異なるが、概念的には次のような複数Layerを含み得る。

```text
Model
├─ Weight-level Alignment / Refusal Tendency
├─ Chat Template / System Policy
├─ Input Filter / Classifier
├─ Output Filter / Classifier
├─ Refusal Logic
├─ Generation Constraint
├─ Runtime Policy
└─ External Guardrail
```

すべてが常に存在するわけではなく、ModelやRuntimeによって構成は異なる。

重要なのは、

> **Safetyは必ずしもModel Weightへ完全統合された一枚岩ではない**

という点である。

外部・Runtime・Prompt・Classifier等として実装されている部分であれば、Model自体を再学習せず変更可能な場合がある。

---

# 4. Instruct ModelでもSafety Stack側を直接変更できる可能性

既存Instruct Modelであっても、ローカルでModelとRuntimeを管理できるのであれば、

* Chat Template
* System Prompt
* Refusal処理
* Runtime Logic
* Classifier
* Pre-processing
* Post-processing
* External Guardrail

等はCodeとして確認・変更できる可能性がある。

現在であればDevelopment AgentへRepositoryを解析させ、

```text
どこでSafety判定しているか
↓
どの処理がRefusalを発生させているか
↓
どのModuleを変更すべきか
↓
変更
↓
Test
```

まで実施させることも可能である。

したがって、

```text
大量の教師Dataを人間が作る
↓
Fine-tuning
```

だけを最初の選択肢とする必要はない。

まず既存Safety Stackの構造を調査し、**再学習なしで目的を達成可能な範囲を確認する方が短い可能性がある。**

---

# 5. Weight側へ焼き込まれたRefusal傾向

ただし、Safety Stackを外部処理だけと仮定してはならない。

Instruct Modelでは、SFT、Preference Optimizationその他のPost-trainingにより、

> 「特定種類の要求へ回答しない」

というBehaviorそのものがWeightへ残っている可能性がある。

したがって、

```text
External Safety Stack OFF
```

にしても、

```text
Model自身がRefusal
```

する可能性がある。

この場合、

* Safety関連Adapterの変更
* Uncensor Adapter
* LoRA等による追加調整
* 別Modelの採用
* 必要部分のみFine-tuning
* その他Model Weight側への変更

などを追加検討する。

基本順序は、

```text
Safety Stackを調査
↓
再学習不要部分を変更
↓
Behavior Test
↓
残存Refusal確認
↓
必要な場合のみWeight-level Adjustment
```

とする。

**最初から人手Dataset作成へ飛ばない。**

---

# 6. Original ModelをImmutable Baselineとして保持する

Uncensored化を行う場合でも、元Modelを破壊的に上書きしない。

```text
Original Model
    = Immutable Baseline
```

として保持する。

目的は以下。

### 6.1 Rollback

変更が失敗した場合に元へ戻れる。

### 6.2 後日追加検証

当初取得しなかった評価項目が後から必要になった場合でも、Original Modelを再度起動して比較できる。

### 6.3 Causalityの追跡

変更後にBehaviorが変化した場合、

```text
元からそうだったのか
変更によって発生したのか
```

を比較できる。

### 6.4 Model Provenance

どのModelを元に、何を変更して現在Versionになったのかを追跡できる。

---

# 7. Uncensored化前にRefusal BehaviorのEvidenceを取得する

変更前Modelについて、可能な限りBaseline Evidenceを取得する。

例えば、

```text
Prompt ID
Category
Original Response
Refusal / Non-refusal
Refusal Pattern
Output Length
Policy Category
Model Version
Model Hash
Runtime Config
```

等を保存する。

目的はSafety研究そのものではなく、

> **業務用Uncensored Modelを作る過程で、何を変更した結果どうBehaviorが変わったかを後から説明・再検証可能にすること**

である。

---

# 8. Development Agentへ変更履歴を残させる

Nazuna Researchで採用しているEvidence-drivenな開発方式を、そのまま利用できる。

Development Agentへ、

```text
何を変更したか
なぜ変更したか
対象File
対象Module
変更前
変更後
Test結果
残存Issue
Maximum Claim
```

を毎回記録させる。

これにより、

```text
Model v0
↓
Change Set 001
↓
Model v1
↓
Change Set 002
↓
Model v2
```

という変更系列を追跡できる。

Uncensored化そのものをブラックボックス的な改造にせず、**Versioned Modificationとして扱う。**

---

# 9. 「一画面に複数Modelを並べて同じPromptを投げる」方式

前職では、一つのUI上に複数ModelをLoadし、同一Promptへ3Modelそれぞれ回答させる仕組みが存在した。

この方式はUncensored Model開発にも相性が良い。

最低構成では、

```text
Same Prompt
   │
   ├─ Original Model
   │
   └─ Uncensored Model
```

を同時実行する。

さらに3枠使用するなら、

```text
Same Prompt
   │
   ├─ A: Original Model
   ├─ B: Current Uncensored Model
   └─ C: Previous Version / Alternative / Evaluator
```

等が考えられる。

---

# 10. Side-by-side比較で得られるもの

単に「回答した／拒否した」を目視するだけでなく、

```text
Original Output
Uncensored Output
Refusal Difference
Category
Model Version
Change Set
Runtime Profile
```

を一緒に保存する。

例えば、

```text
Original:
refused

Uncensored v3:
answered

Change:
refusal_policy disabled
alignment adapter v2
```

と記録できれば、

**どのVersionから業務要件を満たすようになったか**を追跡可能になる。

元Modelを常に横に置けるため、能力劣化も比較できる。

例えば、

```text
拒否は減った
しかしInstruction Followingも悪化した
```

なら、そのUncensored化は失敗と判断できる。

---

# 11. Safety StackをLoad時にON/OFFできる構造

発展案として、

> Safety Stackの各構成要素をModel Load時にProfileとしてON/OFFできる構造

が考えられる。

例えば概念的には、

```yaml
model_profile: internal_uncensored

safety_stack:
  safety_adapter: false
  refusal_policy: false
  input_classifier: false
  output_classifier: false
  external_guardrail: false
```

あるいは、

```yaml
model_profile: standard_internal

safety_stack:
  safety_adapter: true
  refusal_policy: true
  input_classifier: true
  output_classifier: true
  external_guardrail: true
```

といったProfile切替である。

---

# 12. Safety ON/OFF機構の目的

議論途中、一度この構造を

> Safety StackのAblation研究Harness

として発展させる方向へ話がズレた。

研究用途としては成立するが、**本構想の主目的ではない。**

本来の目的は、

> **業務用Uncensored Modelを効率的に使うこと**

である。

したがってSafety ON/OFFは研究目的ではなく、

```text
同じModel基盤を
用途に応じてOperational Profile変更するため
```

の機能として位置付ける。

---

# 13. Uncensored Coreを中心にする構造案

議論では、

> Uncensored ModelをBaseとして、Original Modelとの差分となるSafety機構をLoad時に適用できないか

という案が出た。

概念的には、

```text
Uncensored Core
      +
Optional Safety Components
      ↓
Operational Model
```

である。

例えば、

```text
Uncensored Business Model
├─ Safety Adapter
├─ Refusal Policy
├─ Input Classifier
├─ Output Classifier
├─ External Guardrail
└─ Business-specific Policy
```

を必要に応じて追加する。

これにより、

```text
Guardrail Dataset Generation
→ Uncensored

Internal General Task
→ Standard

Restricted Business Task
→ Restricted

Red Team Data Generation
→ Uncensored + Logging
```

のような切替が可能になる。

---

# 14. Original Modelとの差分をLoad時適用する案

理論上、

```text
Original Model
Uncensored Model
Safety Difference
```

が明確に分離できれば、

> Uncensored ModelへSafety差分をLoad時に追加してOriginalに近いBehaviorへ戻す

構造も考えられる。

ただし、これは常に容易とは限らない。

Safety AlignmentがWeight全体へ分散している場合、

```text
Original - Uncensored
```

が綺麗な小型Safety Moduleになる保証はない。

そのため実装方式としては、

* Adapter
* LoRA
* Low-rank Delta
* Sparse Delta
* Runtime Policy
* External Classifier

等として分離可能なSafety部分をModule化する方が扱いやすい可能性がある。

---

# 15. Canonical BaselineとOperational Baseは分けて考える

運用上、

```text
業務実行のBase
= Uncensored Model
```

とすることは可能である。

しかしProvenance上は、

```text
Canonical Baseline
= Original Model
```

を保持する。

したがって概念的には、

```text
Original Model
    │
    └─ immutable provenance / recovery baseline

Derived Uncensored Business Model
    │
    ├─ generator profile
    ├─ labeler profile
    ├─ reviewer profile
    └─ optional safety profiles
```

という構造になる。

---

# 16. Guardrail学習データ生成へ利用する

Uncensored Modelの第一用途は、Guardrail用Synthetic Data生成である。

通常のSafety-aligned Modelでは生成しづらい、

* Policy violation candidate
* Boundary case
* Adversarial input
* Harmful-looking input
* False-positive candidate
* False-negative candidate
* 表現の言い換え
* 長文
* 婉曲表現
* 複数Turn
* Context-dependent case

等を大量生成できる。

これにより、人間が一件ずつ例文を書く必要を大幅に減らす。

---

# 17. LabelingもLLMで自動化可能

重要な前提として、

> Data GenerationだけでなくLabelingもLLMへ任せられる

という実運用上の経験がある。

前職当時も、汎用LLMへLabelingを行わせていた。

したがって、

```text
LLMはLabelingには使えない
Humanが必須
```

という前提は置かない。

当時の実運用上の問題は、

> 大量・連続・Safety-sensitiveなDataを扱うとSafety機構が介入し、処理を妨げる場合があること

だった。

業務用Uncensored Modelを用意することで、この制約を減らせる可能性がある。

---

# 18. RAGで分類定義を参照させる

Labelerへ単純に、

```text
これは何カテゴリ？
```

と聞くのではない。

Canonicalな分類定義をRAG等で参照させる。

例えば、

```text
Policy Definition
Category Definition
Severity Definition
Exception Rule
Boundary Rule
Labeling Guideline
Historical Correction
```

等を参照させた上で判定する。

つまり、

```text
Input / Output
      +
Canonical Classification Definition
      ↓
Labeler Agent
```

とする。

これにより、Modelの曖昧な一般知識ではなく、**現在有効な組織内定義に基づくLabeling**へ寄せる。

---

# 19. Agentによる自動Data Production Pipeline

最終的にはAgentを使用して、以下を連続実行できる。

```text
Uncensored Generator
        ↓
Synthetic Data
        ↓
Definition RAG
        ↓
Labeler Agent
        ↓
Cross-check Agent
        ↓
Consistency Validation
        ↓
Guardrail Training / Eval
        ↓
Failure Collection
        ↓
Focused Regeneration
        ↓
...
```

これを継続Loopとする。

---

# 20. Cross-checkと整合性確認

単一ModelのLabelをそのままGround Truthとして確定しなくてもよい。

例えば、

```text
Labeler A
Labeler B
Reviewer C
```

等による独立判定を行う。

確認対象は、

* Category一致
* Severity一致
* Definitionとの整合
* Exception適用
* 重複
* Contradiction
* Missing Label
* Invalid Format

等。

不一致が出た場合には再判定を行わせる。

必要に応じてHuman Reviewへ上げることも可能だが、**Human ReviewをLoopの常時必須工程とはしない。**

目的はHumanをData製造機として使うことではなく、

> **定義・基準・例外・難例に集中できる状態へ移すこと**

である。

---

# 21. Guardrailへ投入後もLoopを止めない

Dataを一回作って終わりではない。

```text
Dataset
↓
Guardrail Training
↓
Evaluation
↓
Failure
```

からFailureを回収する。

例えば、

* False Negative
* False Positive
* Boundary Failure
* Unknown Pattern
* New Adversarial Pattern
* Label disagreement

等を取得する。

そのFailureをGenerator側へ戻し、

```text
「このFailure周辺をさらに生成」
```

させる。

これにより、

> **Guardrailが失敗したところほどDataが増える**

というActive-learning的なLoopを作れる。

---

# 22. 最終的な業務Loop

```text
┌─────────────────────────┐
│ Uncensored Business Model │
└────────────┬────────────┘
             ↓
Synthetic Data Generation
             ↓
Definition / Policy RAG
             ↓
Automated Labeling
             ↓
Independent Cross-check
             ↓
Consistency / Coverage Check
             ↓
Guardrail Training
             ↓
Guardrail Evaluation
             ↓
Failure Harvesting
             ↓
Targeted Regeneration
             │
             └──────────────→ Loop
```

---

# 23. Uncensored Modelは単なる「Safety OFF Model」ではない

業務上の完成形では、

> **拒否しないModelを一個作って終わり**

ではない。

必要なのは、

```text
Business-use Uncensored Model
+
Version Control
+
Safety Profiles
+
Evidence
+
Synthetic Data Pipeline
+
Automated Labeling
+
Agent Validation
+
Guardrail Feedback Loop
```

である。

---

# 24. Model Load Profileの利用例

### Guardrail Data Generator

```text
Safety:
minimal / off

Logging:
maximum

Purpose:
adversarial / harmful / boundary data generation
```

### Automated Labeler

```text
Safety:
minimal / off

RAG:
classification canon enabled

Output:
strict structured label
```

### Cross-check Reviewer

```text
Safety:
minimal / off

Input:
data + label + canonical definition

Purpose:
consistency verification
```

### Standard Internal Assistant

```text
Safety:
standard

Business Guardrail:
enabled
```

### Restricted Production Environment

```text
Safety:
strict

Business Guardrail:
strict

External Output:
controlled
```

同一系列のModelを用途別に使い回しやすくなる。

---

# 25. 3-Model UIの業務利用

前職型の3Model比較UIも、このPipelineの補助として有効。

例えば、

```text
Prompt

A: Original Model
B: Uncensored Model
C: Current Candidate / Reviewer
```

を同時表示する。

用途は、

* Uncensored化の確認
* Capability regression確認
* Refusal比較
* Version比較
* Label比較
* Reviewer判定
* Failure分析

等。

これは本体目的ではないが、**業務用Uncensored Modelを効率的に作るDevelopment / Validation Tool**として有効。

---

# 26. 変更Evidence

各変更では最低限、

```text
model_id
base_model
model_hash
change_set_id
changed_component
change_reason
before_behavior
after_behavior
test_set
test_result
known_failure
rollback_target
```

等を残せる。

Development Agentへ自動記録させることで、人間が変更履歴を手書きし続ける必要を減らす。

---

# 27. 元Model保持によるRecovery

元Modelは削除しない。

```text
Current Uncensored Modelが壊れる
↓
Original Modelあり
↓
Change Historyあり
↓
任意Versionへ戻れる
```

だけでなく、

```text
数か月後
「新しいEval Categoryが必要」
↓
Originalを再起動
↓
同じPrompt Set投入
↓
Currentとの比較
```

も可能。

これはResearch上だけでなく、**業務運用・Audit・Regression確認にも有効**。

---

# 28. 前職当時との違い

前職で観測された流れは概ね、

```text
人間が出力例作成
↓
Modelへ学習させる
↓
Uncensored方向へ調整
```

だった。

現在のNazuna Research構想では、

```text
Safety Stackをまず構造解析
↓
変更可能部分を直接変更
↓
必要ならWeight-level Adjustment
↓
Uncensored Business Model作成
↓
AgentでData生成
↓
AgentでLabeling
↓
AgentでCross-check
↓
Guardrailへ投入
↓
FailureからData再生成
```

となる。

したがって、**人間が100件単位でDataを手作業生産する工程そのものをArchitectureから減らす／消すこと**が主要な改善点となる。

---

# 29. 当時のDevelopment Agent利用可能性について

前職当時に現在相当のDevelopment Agentが利用可能だったか、また同等Capabilityを組織内で利用できたかについては確認されていない。

したがって、

> 「当時もこの方式を採用すべきだった」

とは断定しない。

現在はDevelopment Agentを利用できるため、

* Repository解析
* Safety Stack探索
* Patch
* Test
* Change Logging
* Regression Test
* Evidence作成

を大幅に自動化できる、という現在時点での設計差である。

---

# 30. この構想で最も重要な設計思想

最も重要なのは、

> **Uncensored Model自体を目的化しないこと。**

目的は、

```text
大量Dataを作る
↓
正確に分類する
↓
整合性を見る
↓
Guardrailを改善する
↓
FailureからまたDataを作る
```

という業務Loopを高速化すること。

したがって、

```text
Uncensored Model
```

はそのLoopを成立させる**内部Infrastructure**である。

---

# 31. 最終整理

本構想を一行にすると、

> **Original ModelをImmutable Baselineとして保持しつつ、Safety Stackを可能な限り分離・制御可能にした業務用Uncensored Modelを構築し、そのModelをGenerator・Labeler・ReviewerとしてAgent化し、Canonicalな分類定義をRAG参照させながらGuardrail用Dataの生成・Labeling・整合性確認・学習・Failure再生成を継続的に自動化する。**

さらに、

```text
Original Model
+
Versioned Change Evidence
+
Uncensored Business Model
+
Load-time Safety Profile
+
Multi-model Comparison
+
Synthetic Data Generation
+
Definition-aware Labeling
+
Cross-check
+
Guardrail Feedback Loop
```

を一体化することで、

**「大量の入力・出力例を人間が手作業で作る」**

から、

**「Humanは基準・Architecture・例外・最終Authorityを担い、Agent群が大量生産と検証を回す」**

へ移行する。

---

# 32. 現時点のMaximum Claim

現時点では構想段階なので、

> **この方式で必ず高品質なUncensored ModelまたはGuardrail Datasetを構築できる**

まではClaimしない。

一方で、

* 前職でLLMによるLabeling自体を実際に利用していた
* 大量処理ではSafety機構が業務上の制約となった経験がある
* Original Modelを保持し、差分Evidenceを取得する設計が可能
* Nazuna ResearchではDevelopment AgentによるCode変更・Test・Evidence記録を運用している
* Generator → Labeler → Cross-check → Guardrail → Failure再生成というAutomation Loopを設計可能
* Original / Uncensored / Candidate等をSide-by-side比較するValidation Harnessを構築可能
* Safety StackをOperational Profileとして切替可能にできれば、一つのModel系列を複数の社内用途へ展開できる可能性がある

というところまでは、ここまでの議論で成立している。

したがって現時点の正確な位置付けは、

> **Guardrail学習Data製造をHuman-heavyな作業からAgentic Data Production Loopへ移行するための、業務用Uncensored Model基盤構想**

である。

起点となった問題意識は非常に単純である。

> **大量の教師例を人間が手作業で作り続ける前に、Model／Safety Stack／Runtime側を変更し、自動生産可能な構造へした方が早いのではないか。**

そこから、**Model改変 → Evidence → Uncensored業務Model → 自動生成 → 自動Labeling → Cross-check → Guardrail改善 → Failure再投入**までを一つの継続的な業務Architectureとして整理した。
