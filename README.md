# Circulátelo — Bug Reproduction & Sandbox

Sandbox repository to investigate, reproduce, and fix a recursive `MutationObserver` performance and memory leak bug affecting Firefox / Linux on [circulatelo.com](https://circulatelo.com).

## 📁 Repository Structure

```
├── minimal-repro/
│   └── index.html         # Minimal standalone prototype reproducing the feedback loop
├── site-mirror/
│   ├── index.html         # Full mirror of the circulatelo.com homepage
│   ├── robots.txt
│   └── circulatelo-api/   # Mocked JSON endpoints (/api/espacios, /api/stats)
└── README.md
```

---

## 🐛 Bug Overview

### Root Cause
Inside the custom map/search script, a `MutationObserver` was attached to `map.getContainer()` observing all `style` attribute changes in the subtree:

```javascript
const mutationObserver = new MutationObserver(function () {
  allMarkers.forEach(function (item) {
    const el = item.marker.getElement();
    if (el.classList.contains('marker-hidden')) {
      el.style.opacity = '0';          // <-- Triggers 'style' mutation
      el.style.pointerEvents = 'none'; // <-- Triggers 'style' mutation
    }
  });
});

mutationObserver.observe(map.getContainer(), {
  attributes: true,
  subtree: true,
  attributeFilter: ['style']
});
```

Because modifying `el.style` inside the observer callback triggers another mutation event within the observed container, it creates an **unbounded microtask loop**.

---

## 🚀 Running the Sandboxes

### 1. Minimal Prototype (`minimal-repro/`)
A clean single-page demonstration with safety caps to safely inspect the loop counter:

```bash
cd minimal-repro
python3 -m http.server 8080
# Visit http://localhost:8080
```

### 2. Full Site Mirror (`site-mirror/`)
The full site mirror with mocked API data:

```bash
cd site-mirror
python3 -m http.server 8080
# Visit http://localhost:8080
```

---

## 🔧 Recommended Fix

1. Remove the `MutationObserver` completely from the JavaScript bundle.
2. Rely on declarative CSS:

```css
.marker-hidden {
  opacity: 0 !important;
  pointer-events: none !important;
}
```
