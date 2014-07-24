<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
    <title>{{ page.title }}</title>

	<link rel="icon" type="image/x-icon" href="https://identicons.github.com/68227244696026713a09eedd5c1269e0.png">


    {% if page.description %}
    <meta name="description" content="{{ page.description }} ">
    {% else %}
    <meta name="description" content="{{ site.description }} ">
    {% endif %}

    {% if page.categories %}
    <meta name="keywords" content="{{page.categories | join: ',' }}">
    {% else %}
    <meta name="keywords" content="{{site.keywords | join: ',' }}">
    {% endif %}

    {% if page.norobots %}
    <meta name="robots" content="noindex">
    {% endif %}

	<meta name="viewport" content="width=device-width">
	<!-- syntax highlighting CSS -->
    <link rel="stylesheet" href="/css/syntax.css?{{ site.time | date: "%Y%m%d%H%M%S" }}">
    <!-- Site CSS -->
    <link rel="stylesheet" href="/css/main.css?{{ site.time | date: "%Y%m%d%H%M%S" }}">

    <!-- Custom CSS -->
    {% for one_css in page.custom_css %}
    <link rel="stylesheet" href="/css/{{ one_css }}?{{ site.time | date: "%Y%m%d%H%M%S" }}">
    {% endfor %}

</head>