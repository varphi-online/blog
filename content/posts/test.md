+++
date = '2026-04-08T16:41:48-07:00'
draft = true
title = 'I Hate SSR'
+++
![caption or something](attachments/Pasted%20image%2020260408191545.png)

# "Portability" Ruined Everything

$$B=\begin{bmatrix}
C_{a_{1},b_{1}}&\dots&0 \\
\vdots & \ddots &\vdots \\
0 & \dots & C_{a_{n},b_{n}}
\end{bmatrix}=\begin{bmatrix}
a_{1}&-b_{1} &\dots &\dots & 0 &0 \\
b_{1} & a_{1}&\dots&\dots&0&0 \\
\vdots & \vdots & \ddots & \ddots & \vdots & \vdots \\
\vdots & \vdots & \ddots & \ddots & \vdots & \vdots \\
0 & 0 & \dots & \dots & a_{n} & -a_{n} \\
0 & 0 & \dots & \dots & b_{n} & a_{n}
\end{bmatrix}$$

```html
<!DOCTYPE html>
<html lang="{{ site.Language.Locale }}" dir="{{ or site.Language.Direction `ltr` }}">
<head>
  {{ partial "head.html" . }}
  {{ with (templates.Defer (dict "key" "global")) }}
    {{partial "css.html" . }}
  {{ end }}
</head>
<body style="max-width: 40em; margin: 0 auto">
	<script>
    const themes = ["light","dark"];
    window.onload = ()=>{
      const theme = localStorage.getItem('theme');
      document.body.className = theme ? theme : "light";
      document.querySelectorAll(".theme").forEach(t=>t.style = null);
      document.getElementById(document.body.className).style.display = "block";
    }
    function toggleTheme(){
      const lastTheme = document.body.className;
      const nextTheme = themes.at((themes.indexOf(lastTheme) + 1) % themes.length);
      document.body.className = nextTheme;
      localStorage.setItem("theme", nextTheme);
      document.getElementById(lastTheme).style = null;
      document.getElementById(nextTheme).style.display = "block";
    }
	</script>
  <style>
    .theme {
      height: 2em;
      display: none;
    }
  </style>
	<button onclick="toggleTheme()" class="cursor-pointer absolute right-1.25 top-1.25">
		<img id="light" src="/sun.svg" class="theme" style="display: block;">
		<img id="dark" src="/moon.svg" class="theme">
	</button>
  <header class="mb-20">
    {{ partial "header.html" . }}
  </header>
  <main>
    {{ block "main" . }}{{ end }}
  </main>
  <footer>
    {{ partial "footer.html" . }}
  </footer>
</body>
</html>
```

I am so $\begin{bmatrix}a&b & x\\ d\end{bmatrix}$
is a matrix