{{date}} {{time}}
**Clase**: [[]]
**Temas**: 

---



```dataview
TABLE WITHOUT ID
  map(file.inlinks, (x) => link(x)) AS "🔗 Referencias"
WHERE file.name = this.file.name
```