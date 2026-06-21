# Spring Boot Suite — Claude Code Skill

A complete Spring Boot backend development suite for [Claude Code](https://claude.com/claude-code). One skill, three integrated sub-skills:

| Sub-skill | What it does |
|-----------|--------------|
| **springboot-backend** | Production coding conventions + reference library (JPA, Flyway, Spring Security, JUnit 5 + Testcontainers, OpenAPI). |
| **spring-boot-spec-driven** | Docs-first delivery with mandatory **data-model-first** ordering — design before code. |
| **monorepo-cicd** | Full CI/CD pipeline scaffold: GitHub Actions → Argo CD → Kubernetes (AWS EKS) + Terraform. |

Claude Code **hooks** are bundled for file protection (`protect-files.sh`) and auto-formatting (`spotlessApply` on save).

---

## When it triggers

Claude activates this skill automatically whenever you work on a Spring Boot / Java backend — scaffolding a project, designing entities or REST controllers, writing repositories, adding Flyway migrations, configuring Spring Security, writing tests, documenting APIs, implementing a ticket, reviewing/refactoring code, or deploying to Kubernetes.

It also responds to phrases like _"set up monorepo"_, _"CI/CD pipeline"_, _"deploy to Kubernetes"_, _"spec-driven development"_, _"data model first"_, _"docs before code"_, or _"bootstrap a new service"_.

---

## Installation

In Claude Code:

```
/plugin marketplace add shankhanraman/springboot-suite
/plugin install springboot-suite@springboot-suite
```

---

## Requirements

- **Claude Code**
- A Spring Boot repository (Maven or Gradle), **Java 17+**
- Hooks require **bash** and **jq**
- CI/CD features require an **AWS account**, a **GitHub repo**, **Terraform**, and **kubectl**

---

## Repository layout

```
springboot-suite/   (repo root)
├── README.md                          ← you are here
├── .claude-plugin/
│   └── marketplace.json               ← marketplace manifest (lists the plugin)
└── springboot-suite/                  ← the plugin
    ├── .claude-plugin/
    │   └── plugin.json                ← plugin manifest
    ├── hooks/
    │   ├── hooks.json                 ← file-protection + auto-format hooks
    │   └── protect-files.sh
    └── skills/
        └── springboot-suite/
            ├── SKILL.md               ← skill definition + metadata
            ├── references/            ← spring-boot-core, persistence-jpa, flyway-migrations,
            │                            spring-security, spring-boot-testing, api-documentation,
            │                            java-best-practices, monorepo-cicd-{backend,frontend,infra}
            └── assets/templates/      ← architecture.md, datamodel.md, dataflow.md
```

---

## Version

`springboot-suite` v1.0.0 · category: software-development

## License

Add your license of choice (e.g. MIT) here.
```