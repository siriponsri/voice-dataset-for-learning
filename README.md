# README — Voice Dataset for Learning (Qwen3-TTS / LoRA Fine-tuning)

ชุดข้อมูลนี้ถูกสร้างเพื่อใช้ Fine-tune แบบ **LoRA บน Qwen3-TTS** โดยโฟกัส “คุณภาพเสียง” และ “ความพร้อมใช้งานสำหรับ Demo”

---

## 1) วัตถุประสงค์ของชุดข้อมูล
- ให้ทีมสามารถนำไปเทรนได้ทันที (พร้อม manifests + metadata + EDA)
- ลดความสับสนว่าไฟล์ไหนใช้ทำอะไร
- ทำให้ทำซ้ำได้ (reproducible) โดยผูกกับ output ของ NB01

---

## 2) แนวคิดการเทรน (แนะนำ)
### Stage 1 — Thai Anchor Warmup (Optional)
> ถ้าคุณใช้ Stage1 dataset จาก Hugging Face แล้วเจอ auth ให้ login ก่อน  
> `from huggingface_hub import login; login()`

เป้าหมาย: ทำให้โมเดล “นิ่ง” กับภาษาไทย/จังหวะก่อน แล้วค่อยไป Stage 2

### Stage 2 — Target Speaker Fine-Tune (Main)
ใช้ dataset ใน repo นี้เป็นหลัก (multi-speaker clean)

---

## 3) Pipeline ที่ใช้สร้างข้อมูล (NB01)
ภาพรวม:
1) Download/Extract audio → normalize เป็น WAV (mono, 24kHz, PCM16)
2) **Song เท่านั้น** ใช้ **UVR5 STRICT pipeline**
   - MDX (Kim_Vocal_2 / UVR-MDX-NET) → แยก vocals
   - Demucs (htdemucs_ft) → de-bleeding (ลด percussion bleed)
   - DeReverb (UVR-DeEcho-DeReverb) → ทำให้เสียง “dry”
   > หมายเหตุ: บางครั้งจะได้ไฟล์ 2 แบบ (Reverb/No Reverb) ให้เลือกใช้ไฟล์ที่ “มีเสียงจริง” (ส่วนมาก No Reverb)
3) VAD / Silence trimming → ตัดช่วงเงียบออก
4) Chunking → ทำ chunk ความยาว **6–12s** โดยเน้นช่วงที่มีเสียงพูด/ร้องต่อเนื่องจริง
5) Normalization → คุมระดับเสียง + จำกัด peak ลดเสียงแตก
6) Final QC + Clean → ตัด outliers (สั้น/ยาว/เงียบ/clip สูง)
7) Export → แพ็คเป็นโครงสร้าง repo-style (wavs + manifests + metadata + eda)

---

## 4) โครงสร้างโฟลเดอร์ (สิ่งที่ต้องมี)
.
├── wavs/
│ ├── <speaker_id>/*.wav
│ └── ...
├── manifests/
│ ├── train.jsonl
│ └── val.jsonl
├── metadata.csv
└── eda/
└── *.html

---


### 4.1 `wavs/`
ไฟล์เสียงที่พร้อมเทรนแล้ว (mono 24kHz, PCM16) แยกตาม speaker

### 4.2 `manifests/`
ไฟล์ JSONL สำหรับเทรน (1 บรรทัด = 1 sample)

### 4.3 `metadata.csv`
ตาราง metadata สำหรับ audit/EDA เช่น duration, rms, silence_ratio, speaker, language ฯลฯ

### 4.4 `eda/`
ไฟล์รายงานแบบ interactive (Plotly HTML) สำหรับตรวจคุณภาพข้อมูล

---

## 5) Manifest format (Qwen3-TTS)
แต่ละบรรทัดใน `train.jsonl / val.jsonl` มีรูปแบบใกล้เคียง:
```json
{"key":"<utt_id>","text":"<transcript>","audio_path":"wavs/<speaker>/<utt_id>.wav","speaker_id":"<speaker>","language":"<lang>"}

>ข้อควรระวัง: ห้ามย้าย/เปลี่ยนชื่อไฟล์ wav หากไม่ regenerate manifests ใหม่

---

## 6) Checklist ก่อนเทรน (แนะนำให้ทำ)
- เปิด eda/*.html ดู distribution (duration / rms / silence_ratio / clip_ratio)
- สุ่มฟัง wav อย่างน้อย speaker ละ 5–10 ไฟล์
- ตรวจว่า manifests/*.jsonl อ้างถึงไฟล์ที่มีอยู่จริง

---

## 7) วิธีเทรน 
- ใช้ manifests/train.jsonl และ manifests/val.jsonl
- แนะนำเริ่มจาก checkpoint หลัง Stage1 (ถ้าทำ)
- ใช้ fixed test prompts ชุดเดิมทุกครั้ง เพื่อเทียบคุณภาพเสียงระหว่าง epoch

---

8) หมายเหตุเรื่องการแชร์ (สำคัญ)
ชุดข้อมูลนี้มาจากแหล่งสาธารณะ (เช่น YouTube) จึงควรใช้เพื่อการเรียน/งานในทีมตามขอบเขตที่เหมาะสม และหลีกเลี่ยงการเผยแพร่สาธารณะถ้ามีข้อจำกัดด้านลิขสิทธิ์ของแหล่งที่มา