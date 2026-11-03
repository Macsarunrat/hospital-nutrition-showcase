# Ready-to-Use Portfolio Bullet Points (CV / Resume & LinkedIn)

Use these curated bullet points when applying for **DevOps Internship**, **Platform Engineer**, or **Backend Infrastructure** roles. They are framed with active verbs, clear technical ownership, and concrete engineering outcomes.

---

## 📄 For Your English Resume / CV

### Option A: Highlight under "Projects" or "Work Experience"
**Hospital Nutrition & Clinical AI Platform — DevOps & Infrastructure Engineer**  
*(Multi-container production deployment for an institutional university hospital)*

- **Container Orchestration & Architecture:** Architected and deployed an 8-service microservices stack (Django REST, FastAPI, Celery, Redis, PostgreSQL + pgvector, Traefik) using Docker Compose with strict healthcheck-based dependency sequencing and network isolation.
- **Daemon Security Hardening:** Hardened Docker daemon access by integrating `tecnativa/docker-socket-proxy`, eliminating direct `/var/run/docker.sock` exposure to edge proxies and restricting API queries to read-only endpoints.
- **Multi-Tier Subdirectory Routing:** Solved complex path-based reverse proxy routing (`/hospital-nutrition/`) across perimeter Nginx and internal Traefik, synchronizing Vite asset bundling, Django `FORCE_SCRIPT_NAME`, WhiteNoise storage, and HttpOnly cookie path scoping.
- **Automated CI/CD Pipelines:** Built GitHub Actions CI/CD workflows featuring automated Django migration dry-run checks (`makemigrations --check --dry-run`) to prevent runtime schema drift, with zero-downtime Portainer webhook rollouts gated behind protected environment approvals.
- **Asynchronous Task Reliability:** Scaled meal image analysis using Celery and Redis with LRU memory limits, implementing `--max-tasks-per-child=1000` recycling to prevent C-extension memory leaks from image processing libraries.

---

## 🇹🇭 สำหรับเรซูเม่ภาษาไทย (Resume Thai Version)

**วิศวกรระบบและโครงสร้างพื้นฐาน (DevOps & Backend Infrastructure) — ระบบโภชนาการและวิเคราะห์อาหารโรงพยาบาล**

- **การจัดการโครงสร้างพื้นฐานด้วย Docker:** ออกแบบและบริหารจัดการระบบ Multi-container 8 เซอร์วิส (Django REST, FastAPI, Celery, Redis, PostgreSQL, Traefik) บน Docker Compose พร้อมตั้งค่า Healthchecks และ Network Isolation ป้องกันฐานข้อมูลรั่วไหล
- **ยกระดับความปลอดภัย (Security Hardening):** เสริมความปลอดภัยของ Docker Host โดยใช้ `docker-socket-proxy` กั้นระหว่าง Traefik กับ Docker Daemon จำกัดสิทธิ์เฉพาะการอ่าน (Read-only) ป้องกันการโจมตีเจาะระบบถึงระดับ Root
- **แก้ปัญหาระบบ Routing สถาบัน:** แก้ปัญหาการ Deploy แบบ Subdirectory (`/hospital-nutrition/`) เชื่อมต่อระหว่าง Nginx ของสถาบันกับ Traefik แก้ไขปัญหา URL Prefix และ Cookie Scoping ของระบบ Authentication ให้ทำงานได้อย่างแม่นยำ
- **ระบบ CI/CD อัตโนมัติ:** พัฒนา GitHub Actions Pipeline ตรวจสอบความถูกต้องของ Database Migration อัตโนมัติก่อนรวมโค้ด และสั่ง Deploy ขึ้น Production อัตโนมัติผ่าน Webhook ของ Portainer พร้อมระบบอนุมัติ (Approval Gate)
- **เพิ่มความเสถียรของ Background Worker:** วางระบบ Celery Worker ประมวลผลภาพอาหารคู่กับ Redis โดยตั้งค่า Recycle Process (`--max-tasks-per-child`) ป้องกันปัญหา Memory Leak จาก Library ประมวลผลภาพ

---

## 🌐 For Your LinkedIn Profile (Projects or Featured Section)

**Project Title:** Hospital Nutrition & Clinical AI Platform — Production DevOps Infrastructure  
**Associated with:** Self-directed / Academic / Clinical Project  
**Skills:** Docker, Docker Compose, Traefik, GitHub Actions, CI/CD, PostgreSQL, Redis, Celery, Linux, Nginx, Information Security.

**Description:**  
*Engineered and deployed the full production infrastructure for a clinical hospital nutrition system operating on institutional infrastructure.*

*Key Engineering Achievements:*
- *Orchestrated 8 isolated services using Docker Compose, establishing healthcheck boot ordering and internal network boundaries.*
- *Eliminated Docker host escalation risks by sandboxing reverse proxy daemon access with Docker Socket Proxy.*
- *Overcame enterprise subdirectory routing constraints across institutional Nginx, Traefik, and Django application layers.*
- *Automated integration testing and migration dry-runs using GitHub Actions, paired with protected webhook deployments to Portainer.*

---

## 💬 Talking Points in an Interview (แนวทางตอบคำถามในห้องสัมภาษณ์)

เมื่อผู้สัมภาษณ์ถาม: *"ช่วยเล่าสิ่งที่คุณภูมิใจที่สุดหรือปัญหาที่ยากที่สุดในโปรเจกต์นี้ให้ฟังหน่อย?"*

**แนวทางการตอบ:**
1. **เล่าโจทย์และขอบเขตงาน:**  
   *"ในโปรเจกต์นี้ ผมรับผิดชอบงานฝั่ง Backend, Database และ Infrastructure ทั้งหมดตั้งแต่ต้นจนนำขึ้นรันจริงบนเซิร์ฟเวอร์ของโรงพยาบาลมหาวิทยาลัยครับ"*
2. **หยิบปัญหา Subdirectory Routing มาเล่า (แสดงทักษะ Full-stack & Networking):**  
   *"ความท้าทายแรกคือสถาบันไม่อนุญาตให้สร้าง Subdomain แยก แต่บังคับให้รันใต้ Subdirectory `/hospital-nutrition/` ซึ่งทำให้เจอปัญหา Browser ตัด Path ทิ้ง ผมต้องแก้ปัญหาแบบ End-to-end ตั้งแต่ Nginx ตัวหน้า, Traefik ที่ทำ Strip Prefix, Vite ที่ต้องฝัง Base Path, และ Django `FORCE_SCRIPT_NAME` ตลอดจน Cookie Path เพื่อให้ JWT ส่งถูกต้อง"*
3. **หยิบเรื่องความปลอดภัยมาเล่า (แสดง Mindset ด้าน Security ที่เหนือกว่าเด็กทั่วไป):**  
   *"อีกเรื่องคือ Security ทั่วไปมักจะแนะนำให้ mount `/var/run/docker.sock` เข้า Traefik ตรงๆ ซึ่งอันตรายมากเพราะเท่ากับให้ Root สิทธิ์บนโฮสต์ ผมจึงวาง `docker-socket-proxy` มาคั่นไว้ และเปิดสิทธิ์เฉพาะ Endpoint ที่จำเป็นต้องใช้จริงๆ"*
