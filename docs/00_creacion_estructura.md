
# Creación de la estructura inicial del repositorio DatAlive

## Paso 1. Crear la estructura de carpetas en la PC
Desde **CMD en Windows**, dentro de la carpeta del repositorio clonado (`DatAlive/`):

```cmd
mkdir awsDA\evidencias
mkdir azureDA\evidencias
mkdir gcpDA\evidencias
mkdir common
mkdir docs
````

## Paso 2. Crear archivos .gitkeep para que GitHub reconozca las carpetas vacías

GitHub no guarda carpetas vacías, por lo tanto se usan archivos **.gitkeep**:

```cmd
echo. > awsDA\.gitkeep
echo. > awsDA\evidencias\.gitkeep
echo. > azureDA\.gitkeep
echo. > azureDA\evidencias\.gitkeep
echo. > gcpDA\.gitkeep
echo. > gcpDA\evidencias\.gitkeep
echo. > common\.gitkeep
echo. > docs\.gitkeep
```

## Paso 3. Subir cambios a GitHub

Se agregan los archivos al control de versiones y se suben al repositorio remoto:

```bash
git add .
git commit -m "Estructura inicial de carpetas con .gitkeep y actualización de README"
git push origin main
```

## Resultado

La estructura base quedó creada en GitHub con las siguientes carpetas:

```
awsDA/
azureDA/
gcpDA/
common/
docs/
```
