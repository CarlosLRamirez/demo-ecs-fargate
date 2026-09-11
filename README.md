# Demo: Amazon ECS con Fargate

Lab/demo para correr una aplicación de muestra en un clúster de Amazon ECS con
AWS Fargate, expuesta a través de un Application Load Balancer.

## Estructura del repositorio

```
.
├── cloudformation/
│   └── vpc-base.yaml        # VPC, subredes y security groups base
├── student/
│   └── Lab_Alumno_ECS_Fargate.md
└── instructor/                # No se sube al repo (ver .gitignore)
    └── Demo_ECS_Guia_Instructor.md
```

## Qué crea el CloudFormation

La plantilla `cloudformation/vpc-base.yaml` crea únicamente la infraestructura
de red base:

- Una VPC
- Dos subredes públicas en dos zonas de disponibilidad distintas
- Internet Gateway y tabla de rutas
- Security group para el Application Load Balancer (permite HTTP desde internet)
- Security group para las tasks de Fargate (permite tráfico solo desde el ALB)

**No crea** el clúster ECS, la task definition, el service, ni el ALB — eso se
hace paso a paso siguiendo la guía correspondiente.

## Cómo usar este repo

- Si eres alumno, sigue `student/Lab_Alumno_ECS_Fargate.md` de principio a fin.
- Si eres instructor, la guía de instructor (no incluida en este repo público)
  tiene el guion para desplegar el CloudFormation de antemano y correr la demo
  en vivo con el tiempo estimado por paso.

## Limpieza

Ambas guías incluyen instrucciones de limpieza al final. El Load Balancer y
las tasks de Fargate generan costo mientras están activos — no lo dejes
corriendo sin necesidad.
