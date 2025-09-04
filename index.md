---
title: Übungen für Azure-Entwickelnde
permalink: index.html
layout: home
---

## Übersicht

Die folgenden Übungen sind darauf ausgelegt, Ihnen eine praktische Lernerfahrung zu bieten, in der Sie allgemeine Aufgaben erkunden, die Entwickelnde beim Erstellen und Bereitstellen von Lösungen in Microsoft Azure ausführen.

> **Hinweis**: Um die Übungen durchzuführen, benötigen Sie ein Azure-Abonnement mit ausreichenden Berechtigungen und Kontingenten, um die erforderlichen Azure-Ressourcen bereitzustellen. Wenn Sie noch kein Konto haben, können Sie sich für ein [Azure-Konto](https://azure.microsoft.com/free) anmelden. 

Einige Übungen können zusätzliche oder weitere Anforderungen beinhalten. Diese enthalten einen Abschnitt **Bevor Sie beginnen**, der für diese Übung spezifisch ist.

## Themenbereiche
{% assign exercises = site.pages | where_exp:"page", "page.url contains '/instructions'" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %}

<ul>
{% for group in grouped_exercises %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in grouped_exercises %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }} 

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">Zurück zum Anfang</a> {% endfor %}

