---
layout: page
---

Welcome to my webpage. 


<table>
	<tr>
	{% for social in site.social %}
	<td>
		<a href="{{ social.url }}" class="btn btn-default btn-lg"><i class="fa fa-{{ social.title }} fa-fw"></i> <span class="network-name">{{ social.title }}</span></a>
	</td>
	{% endfor %}
	</tr>	
</table>