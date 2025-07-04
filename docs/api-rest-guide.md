# MonkeyOCR - Guide API REST

## Déploiement

### Lancer l'API REST

```bash
# Démarrage simple
docker run -p 7861:7861 monkeyocr-fix fastapi

# Démarrage en mode daemon
docker run -d -p 7861:7861 --name monkeyocr-api monkeyocr-fix fastapi

# Avec variables d'environnement
docker run -d -p 7861:7861 -e MONKEYOCR_CONFIG=config.yaml monkeyocr-fix fastapi
```

## Endpoints

### 1. Vérification de santé
```bash
GET /health
```

**Exemple :**
```bash
curl http://localhost:7861/health
```

**Réponse :**
```json
{
  "status": "healthy",
  "model_loaded": true
}
```

### 2. Extraction de texte
```bash
POST /ocr/text
```

**Exemple :**
```bash
curl -X POST "http://localhost:7861/ocr/text" \
  -F "file=@document.pdf"
```

**Réponse :**
```json
{
  "success": true,
  "task_type": "text",
  "content": "Contenu extrait du document...",
  "message": "Text extraction completed successfully"
}
```

### 3. Extraction de formules mathématiques
```bash
POST /ocr/formula
```

**Exemple :**
```bash
curl -X POST "http://localhost:7861/ocr/formula" \
  -F "file=@document.pdf"
```

**Réponse :**
```json
{
  "success": true,
  "task_type": "formula",
  "content": "$$\\int_0^1 x^2 dx = \\frac{1}{3}$$",
  "message": "Formula extraction completed successfully"
}
```

### 4. Extraction de tableaux
```bash
POST /ocr/table
```

**Exemple :**
```bash
curl -X POST "http://localhost:7861/ocr/table" \
  -F "file=@document.pdf"
```

**Réponse :**
```json
{
  "success": true,
  "task_type": "table",
  "content": "| Colonne 1 | Colonne 2 |\n|-----------|-----------|",
  "message": "Table extraction completed successfully"
}
```

### 5. Analyse complète de document
```bash
POST /parse
```

**Exemple :**
```bash
curl -X POST "http://localhost:7861/parse" \
  -F "file=@document.pdf"
```

**Réponse :**
```json
{
  "success": true,
  "message": "PDF parsing completed successfully",
  "output_dir": "/tmp/monkeyocr_parse_xxx",
  "files": ["document.md", "document_content_list.json"],
  "download_url": "/static/document_parsed_1234567890.zip"
}
```

### 6. Analyse avec séparation par pages
```bash
POST /parse/split
```

**Exemple :**
```bash
curl -X POST "http://localhost:7861/parse/split" \
  -F "file=@document.pdf"
```

**Réponse :**
```json
{
  "success": true,
  "message": "PDF parsing (with page splitting) completed successfully",
  "output_dir": "/tmp/monkeyocr_parse_xxx",
  "files": ["page_0/document.md", "page_1/document.md"],
  "download_url": "/static/document_split_1234567890.zip"
}
```

### 7. Téléchargement des résultats
```bash
GET /download/{filename}
```

**Exemple :**
```bash
curl -O "http://localhost:7861/download/document_parsed_1234567890.zip"
```

## Formats supportés

- **PDF** : `.pdf`
- **Images** : `.jpg`, `.jpeg`, `.png`

## Intégration Python

### Exemple basique

```python
import requests

def extract_text(file_path):
    with open(file_path, 'rb') as f:
        response = requests.post(
            'http://localhost:7861/ocr/text',
            files={'file': f}
        )
    
    if response.status_code == 200:
        return response.json()['content']
    else:
        raise Exception(f"Erreur API : {response.status_code}")

# Utilisation
text = extract_text('document.pdf')
print(text)
```

### Exemple avec analyse complète

```python
import requests

def parse_document(file_path):
    with open(file_path, 'rb') as f:
        response = requests.post(
            'http://localhost:7861/parse',
            files={'file': f}
        )
    
    if response.status_code == 200:
        result = response.json()
        print(f"Analyse réussie : {result['message']}")
        return result['download_url']
    else:
        raise Exception(f"Erreur API : {response.status_code}")

# Utilisation
download_url = parse_document('document.pdf')
print(f"Télécharger : http://localhost:7861{download_url}")
```

### Exemple avec gestion d'erreur

```python
import requests

def safe_ocr_request(endpoint, file_path):
    try:
        with open(file_path, 'rb') as f:
            response = requests.post(
                f'http://localhost:7861/{endpoint}',
                files={'file': f}
            )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Erreur de requête : {e}")
        return None
    except Exception as e:
        print(f"Erreur : {e}")
        return None

# Utilisation
result = safe_ocr_request('ocr/text', 'document.pdf')
if result and result['success']:
    print(result['content'])
```

## Configuration

### Variables d'environnement

| Variable | Description | Défaut |
|----------|-------------|--------|
| `MONKEYOCR_CONFIG` | Chemin vers le fichier de configuration | `model_configs.yaml` |
| `FASTAPI_HOST` | Host de l'API | `0.0.0.0` |
| `FASTAPI_PORT` | Port de l'API | `7861` |
| `TMPDIR` | Répertoire temporaire | `/tmp` |
| `CUDA_VISIBLE_DEVICES` | Configuration GPU | - |

### Exemple avec configuration personnalisée

```bash
docker run -d \
  -p 7861:7861 \
  -e MONKEYOCR_CONFIG=/app/custom_config.yaml \
  -e TMPDIR=/app/tmp \
  -v $(pwd)/custom_config.yaml:/app/custom_config.yaml \
  -v $(pwd)/tmp:/app/tmp \
  monkeyocr-fix fastapi
```

## Gestion des erreurs

### Codes d'erreur

| Code | Description |
|------|-------------|
| `400` | Type de fichier non supporté |
| `404` | Fichier non trouvé |
| `500` | Erreur interne du serveur |

### Exemple de réponse d'erreur

```json
{
  "success": false,
  "task_type": "text",
  "content": "",
  "message": "OCR task failed: Unsupported file type"
}
```

## Troubleshooting

### Vérification de l'état

```bash
# Vérifier que l'API répond
curl http://localhost:7861/health

# Vérifier les logs
docker logs monkeyocr-api
```

### Problèmes courants

1. **Port déjà utilisé**
   ```bash
   docker run -p 8861:7861 monkeyocr-fix fastapi
   ```

2. **Modèle non initialisé**
   - Vérifier les logs du container
   - S'assurer que la configuration est valide

3. **Fichier trop volumineux**
   - Augmenter la taille du TMPDIR
   - Vérifier les limites Docker