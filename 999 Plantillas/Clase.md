{{date}} {{time}}
**Clase**: [[]]
**Temas**: 

**Estado**: #baby 

---



```dataview
TABLE WITHOUT ID
  map(file.inlinks, (x) => link(x)) AS "🔗 Referencias"
WHERE file.name = this.file.name
```