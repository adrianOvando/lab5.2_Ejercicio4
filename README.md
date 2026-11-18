# Laboratorio 5.2 - Practica Individual: Pipeline de Despliegue Continuo (CD) con Docker

**Universidad San Francisco Xavier de Chuquisaca**  
**Facultad de Tecnologia**  
**Carrera:** Ingenieria en Ciencias de la Computacion  
**Asignatura:** Trabajando en la Nube (COM610)  
**Estudiante:** Jesus Adrian Ovando  
**Docente:** Ing. Marcelo Quispe Ortega  
**Semestre:** 2/2026  

---

## Descripcion del Proyecto

Este repositorio contiene la implementacion completa de un pipeline de **Despliegue Continuo (CD)** utilizando **Docker** y **GitHub Actions** para una API RESTful de gestion de productos desarrollada con Node.js y Express.

El flujo automatizado incluye:
- Compilacion multi-etapa (*multi-stage build*) con `node:20-alpine`.
- Publicacion automatica en Docker Hub (`ciscoad/lab5-2-ejercicio4`) con etiquetas mutables (`latest`) e inmutables asociadas al commit SHA (`${{ github.sha }}`).
- Analisis de vulnerabilidades automatizado con **Aqua Security Trivy**.
- Despliegue remoto automatizado via SSH a una instancia de **AWS EC2** con IP Elastica fija (`18.213.236.16`).
- Validacion en caliente mediante **Health Check** en puerto temporal `3001` previo a la publicacion en el puerto `80`.

---

## Endpoints de la API en Produccion

La aplicacion se encuentra desplegada y accesible en la nube a traves de las siguientes URLs publicas:

- **Sonda de Salud:** http://18.213.236.16/health
- **Catalogo de Productos (CRUD):** http://18.213.236.16/api/products

---

## Documentacion del Laboratorio

Puedes encontrar el informe completo con explicaciones tecnicas, capturas de pantalla de cada etapa y evidencias detalladas en el siguiente enlace:

- [Ver informe del laboratorio](./INFORME.md)
