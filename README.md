# ConceptCraft AI

ConceptCraft AI is a modular, AI-driven adaptive learning platform designed to deliver structured, practice-first education for K–12 students.

The system combines Retrieval-Augmented Generation (RAG) with a deterministic rule-based adaptive engine to ensure reliable, mastery-based learning progression.

Unlike generic AI assistants, ConceptCraft AI separates explanation generation from progression control, enabling structured and measurable skill development.

---

## 🚀 Core Philosophy

**AI generates explanations.  
Deterministic logic controls progression.**

---

## 🏗 System Architecture Overview

The platform follows a layered architecture:

### 1️⃣ Frontend Layer
- React (Single Page Application)
- Interactive practice interface
- Visual explanation rendering (SVG / Canvas)
- Local session state management

### 2️⃣ Backend & Application Layer
- Node.js + Express.js API
- Rule-Based Adaptive Engine
- Performance signal tracking (accuracy, time, hint usage)
- Difficulty selection logic

### 3️⃣ AI & Intelligence Layer
- Cloud-hosted Large Language Model (LLM)
- Retrieval-Augmented Generation (RAG) wrapper
- Structured prompt engineering
- JSON-based response parsing

### 4️⃣ Data Layer
- **MySQL (Structured Curriculum Database)**
  - Concepts
  - Grade levels
  - Difficulty layers
  - Pattern rules
- **NoSQL Document Store**
  - Long-form explanations
  - Knowledge references

### 5️⃣ Cloud Infrastructure
- AWS cloud deployment
- Backend compute services
- Managed database services

---

## 🔄 Learning Flow

1. Student attempts an exercise  
2. Answer is validated deterministically  
3. If incorrect:
   - RAG retrieves relevant structured content
   - LLM generates explanation (structured JSON output)
4. Performance signals are updated  
5. Rule-based adaptive engine selects the next exercise  

> The LLM does **not** control progression.

---

## 🎯 Key Features

- Practice-first learning model  
- AI-driven misconception analysis  
- RAG-based knowledge retrieval  
- Deterministic adaptive progression  
- Difficulty-layered mastery  
- Concept-tagged curriculum structure  
- Modular and scalable architecture  

---

## 💡 Why ConceptCraft AI?

Most AI education systems are either:

- Static learning platforms with fixed progression  
- Or general-purpose AI assistants without structured mastery control  

ConceptCraft AI introduces a structured hybrid model where:

- RAG ensures domain-constrained accuracy  
- A rule-based engine ensures deterministic progression  
- AI enhances understanding without replacing system control  

---

## 📊 Pilot Deployment Model

Designed for city-level deployment across 15–20 schools.

**Estimated Monthly Operational Cost:**  
₹55,000 – ₹90,000  
(~₹35–₹45 per student per month)

### Cost Optimization Strategy
- LLM triggered only on incorrect responses  
- Rule-based engine reduces unnecessary AI calls  
- Caching of frequent explanations  

---

## 🔮 Future Scope

- Multilingual support  
- Expanded curriculum domains  
- Persistent progress tracking  
- Advanced analytics dashboard  
- Knowledge tracing enhancements  

---

## 🛠 Tech Stack

- Frontend: React  
- Backend: Node.js, Express.js  
- AI Layer: Cloud-hosted LLM + RAG  
- Database: MySQL + NoSQL  
- Cloud: AWS  

---

## 📌 Hackathon Submission

This project was developed as part of the **AWS AI for Bharat Hackathon**, focusing on structured, scalable, and responsible AI integration in education.

---

## 📄 License

This repository is part of a hackathon submission.  
Further licensing details may be added in future releases.
