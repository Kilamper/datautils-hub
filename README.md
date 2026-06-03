# DataUtils Hub — Extraction Engine

DataUtils Hub is a modern Vue 3 + Vite web utility designed to extract, clean, and validate structural identifiers from raw text blocks. Built with robust regex engines and official validation algorithms, the application can extract data seamlessly even when inputs are completely chained together without whitespace, delimiters, or line breaks.

## 🚀 Key Features

### 1. Advanced Extraction Engine
- **Chasis / VIN Extraction:** Extracts standard 17-character alphanumeric Vehicle Identification Numbers (excluding `I`, `O`, `Q`) from any text block—including raw continuous strings.
- **Spanish License Plates:** Extracts and separates Spanish vehicle registration plates (4 digits followed by 3 letters, excluding vowels and invalid characters: `[BCDFGHJKLMNPRSTVXYZ]`). Supports chained strings like `1234BBB5678CCC` and strips separators automatically.
- **Custom Regex Extraction:** Allows power users to input custom regular expressions for matching domain-specific records.

### 2. Strict Spanish ID Verification (NIF, NIE, CIF)
Unlike generic regex matchers, the engine validates all extracted DNI, NIE, and CIF tokens mathematically:
- **DNI/NIF (Personas Físicas):** Validates 8 digits + 1 control letter using the official Modulus 23 checksum.
- **NIE (Extranjeros):** Validates foreign IDs starting with `X`, `Y`, or `Z` by mapping them to numerical prefixes (`0`, `1`, `2` respectively) and performing the Modulus 23 check.
- **CIF (Empresas/Entidades):** Validates company numbers via the official CIF checksum algorithm (weighting odd and even digit positions, validating matching control codes against digits or characters `A-J` according to the organization type).

---

## 🛠️ Tech Stack & Setup

This application is built with:
- **Framework:** Vue 3 (Composition API, `<script setup>`)
- **Build Tool:** Vite
- **Styling:** CSS & TailwindCSS

### Installation

Install dependencies:
```bash
npm install
```

### Development Server

Run the local dev server:
```bash
npm run dev
```

### Production Build

Compile and minify the project for production:
```bash
npm run build
```

---

## 📐 Algorithmic Validation Details

### DNI / NIE Modulus 23
The control letter for a DNI or NIE is verified using the remainder of the numerical part divided by 23:
$$\text{Index} = \text{Numerical Part} \pmod{23}$$
The resulting index corresponds to the letter in the standard alphabet chart:
`T R W A G M Y F P D X B N J Z S Q V H L C K E`

### CIF Checksum Validation
Given an organization type letter, 7 digits ($d_1 d_2 d_3 d_4 d_5 d_6 d_7$), and a control character:
1. **Even Sum ($A$):** Sum of digits at even positions: $A = d_2 + d_4 + d_6$.
2. **Odd Sum ($B$):** Double each digit at odd positions ($d_1, d_3, d_5, d_7$). If a doubled value is $> 9$, sum its digits (e.g., $7 \times 2 = 14 \rightarrow 1+4 = 5$). Add these up to get $B$.
3. **Total Sum ($C$):** $C = A + B$.
4. **Control Value ($E$):** $E = (10 - (C \pmod{10})) \pmod{10}$.
5. Matches are checked against $E$ (as a digit) or `"JABCDEFGHI"[E]` (as a letter) based on the organization prefix.

---

## 🧪 Test Case Compatibility

Paste this continuous block of glued text into the input field to test VIN extraction:
```text
SJNF16FA8U2192837SJNF16FA9U2187548SJNJ12TA9U2163337SJNJ12TA5U2163352SJNJ12TA7U2171565SJNJ12TA5U2172682SJNF16FA1U2122158
```
The engine will successfully extract exactly 7 distinct 17-character chassis numbers.
