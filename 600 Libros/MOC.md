---
aliases:
  - 📚 Books
---
```meta-bind-button
label: Nuevo Libro
icon: ""
style: default
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: templaterCreateNote
    templateFile: 999 Plantillas/Books.md
    folderPath: 600 Libros/Notas
    fileName: ""
    openNote: true
    openIfAlreadyExists: false

```
# Libros
```dataview
TABLE Fecha, Titulo, Autor
FROM "600 Libros/Notas"
SORT file.name DESC
```