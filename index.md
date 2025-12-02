---
title: DevOps 実践者とプラットフォーム エンジニア向け演習
description: DevOps、DevSecOps、SRE、プラットフォーム エンジニアリング向け実践演習
permalink: index.html
layout: home
---

# 概要

以下の演習は、DevOps 実践者とプラットフォーム エンジニアが Azure でソリューションを構築するときに実行する一般的なタスクを確認できる実践的な学習経験を提供するために設計されています。

> **注**: これらの演習を完了するには、必要な Azure リソースをプロビジョニングするのに十分なアクセス許可とクォータがある Azure サブスクリプションが必要です。 まだお持ちでない場合は、[Azure アカウント](https://azure.microsoft.com/free)にサインアップできます。

一部の演習には、追加または特定の要件が含まれる場合があります。 これらには、その演習に固有の「**開始する前に**」セクションが含まれます。

## トピック レベル

{% assign exercises = site.pages | where_exp:"page", "page.url contains '/Instructions'" %} {% assign exercises = exercises | where_exp:"page", "page.lab.topic != null" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %} {% assign topic_order = "Basic,Intermediate,Advanced" | split: "," %} {% assign sorted_groups = "" | split: "" %} {% for topic in topic_order %} {% assign matching_group = grouped_exercises | where: "name", topic | first %} {% if matching_group %} {% assign sorted_groups = sorted_groups | push: matching_group %} {% endif %} {% endfor %}

<ul>
{% for group in sorted_groups %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in sorted_groups %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }}

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">トップに戻る</a> {% endfor %}
