### mamabench composition

- **MCQ set:** OBGYN + Pediatrics subsets of MedMCQA (native `subject_name`), filtered MedQA (keyword + tag), filtered MMLU-medical, PediatricsMQA (neonates, infants), PedMedQA (neonates 0–3 months, infants).
- **Open-ended set:** HealthBench items tagged with pregnancy (Z33.1), obstetric (O-chapter), or perinatal (P-chapter) ICD-10 codes, with HealthBench's physician-written rubrics.
- **Safety set:** EquityMedQA obstetric items, FairMedQA maternal items, plus MedEqualQA and MedFuzz perturbations applied to the MCQ anchor set.