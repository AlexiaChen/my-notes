

# my-notes
Una colección de mis notas

```bash
# - **La `/` al final de la ruta es importante**:`/source/` indica sincronizar el contenido del directorio, `/source` indica sincronizar el directorio en sí mismo
rsync -av --delete --progress --exclude='.git/' /home/user/source/ /home/user/backup/
```
