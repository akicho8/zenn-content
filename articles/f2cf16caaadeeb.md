---
title: "Magitでコミットメッセージのプレフィクス入力を自動化する"
emoji: "🐟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Emacs", "Magit", "Git", "yasnippet"]
published: true
---

![](/images/f2cf16caaadeeb/emacs_ss1.png)
*git commit すると即選択画面へ*

![](/images/f2cf16caaadeeb/emacs_ss2.png)
*そこで d ENTER するとこうなる*

## 手順1. yasnippet で展開できるようにしておく

`[` `TAB` で選択肢が出るようにします

```text:~/.emacs.d/snippets/text-mode/[
# --
`(yas-choose-value '(
"[feat] "
"[fix] "
"[refactor] "
"[chore] "
"[style] "
"[docs][ci skip] Update README"
"[test] "
"[perf] "
))`$0
```

## 手順2. 入力モードに入ると自動的に選択肢を出す

```elisp
(defun my-git-commit-prefix-select ()
  "何も入力されていなければ [ を入力し展開する
再編集時を考慮して [ があれば ] の後まで移動する"
  (interactive)
  (if (char-equal (char-after) (string-to-char "["))
      (search-forward "] " nil t)
    (when (or (not (char-after))
              (char-equal (char-after) (string-to-char "\n")))
      (insert "[")
      (yas-expand))))

(add-hook 'git-commit-setup-hook 'my-git-commit-prefix-select)
```

そんだけです
