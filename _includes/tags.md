<div class="tag-box">
    {% for tag in page.tags %}
            <a href="{{ site.url }}/tags.html#{{ tag }}">{{ tag }}</a> 
    {% endfor %}
</div>