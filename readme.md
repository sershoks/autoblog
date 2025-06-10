Absolument. On passe en mode 100% souverain, 100% local, 100% sur votre CPU. Aucun appel externe, aucune clé API payante (sauf pour vos propres services comme WordPress).

C'est un projet avancé mais extrêmement gratifiant. Voici le guide complet, étape par étape.

### Philosophie et Attentes

*   **Souveraineté Totale :** Aucune de vos données ou de vos requêtes ne quitte votre serveur. Vous contrôlez tout.
*   **Coût Zéro (Logiciel) :** Tous les outils sont open source. Le seul coût est celui de votre serveur et de l'électricité.
*   **Compromis sur la Vitesse :** C'est le point crucial. **Attendez-vous à une exécution lente.** Chaque étape impliquant le LLM prendra du temps sur un CPU. La génération complète d'un article peut facilement prendre 10 à 20 minutes, voire plus. La patience est votre meilleure alliée.

---

### Architecture Globale 100% Locale

| Composant | Rôle | Technologie Utilisée |
| :--- | :--- | :--- |
| **Serveur d'IA** | Fait tourner le modèle de langage | **Ollama** |
| **Moteur de Recherche** | Permet de chercher sur le web | **SearxNG** (méta-moteur auto-hébergé) |
| **Cerveau / Orchestrateur** | Définit et exécute les agents | **CrewAI** (Script Python) |
| **Base de Données Contenu** | Stocke et publie les articles | **Votre propre site WordPress** |
| **Alerte / Révision** | Reçoit les articles rejetés | **Votre propre endpoint de Webhook** |

---

### Étape 1 : Installation des Fondations (Ollama & SearxNG)

C'est la partie la plus technique. Nous allons utiliser Docker pour simplifier l'installation de SearxNG.

**1. Installez Docker et Docker Compose sur votre serveur**
Si ce n'est pas déjà fait, suivez les instructions officielles pour votre distribution Linux. C'est un prérequis indispensable.

**2. Installez Ollama**
```bash
# Télécharge et exécute le script d'installation officiel
curl -fsSL https://ollama.com/install.sh | sh
```

**3. Téléchargez et lancez le modèle LLM local**
Nous allons utiliser `llama3:8b`, un excellent compromis pour le CPU.
```bash
# Cette commande télécharge le modèle (plusieurs Go) et le rend disponible
ollama run llama3:8b
```
Une fois le modèle téléchargé, vous pouvez arrêter le processus (`/bye`). Ollama continuera de tourner en service d'arrière-plan, prêt à recevoir des requêtes.

**4. Installez et configurez le moteur de recherche local SearxNG**
Nous utilisons `docker-compose` car c'est la méthode la plus propre.
Créez un dossier pour votre projet, par exemple `~/autoblog`.
Dans ce dossier, créez un fichier nommé `docker-compose.yml` et collez-y le contenu suivant :

```yaml
# docker-compose.yml
version: '3.8'

services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: always
    ports:
      # Expose SearxNG sur le port 8080 de votre serveur
      - "8080:8080"
    volumes:
      # Stocke la configuration et les données de SearxNG de manière persistante
      - ./searxng:/etc/searxng
    environment:
      # Vous pouvez définir une instance publique si vous le souhaitez, mais pour un usage local, ce n'est pas nécessaire
      - SEARXNG_BASE_URL=http://localhost:8080/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
```

Lancez SearxNG :
```bash
# Placez-vous dans le dossier contenant le fichier docker-compose.yml
cd ~/autoblog

# Lancez le service en arrière-plan
docker-compose up -d
```
Attendez une minute, puis vérifiez que tout fonctionne en visitant `http://VOTRE_IP_SERVEUR:8080` dans votre navigateur. Vous devriez voir l'interface de SearxNG.

---

### Étape 2 : Préparation de l'Environnement Python

```bash
# Créez et activez un environnement virtuel (bonne pratique)
python3 -m venv venv
source venv/bin/activate

# Installez toutes les librairies nécessaires
pip install crewai requests beautifulsoup4 langchain-community python-dotenv
```

---

### Étape 3 : Le Code - Les Outils Personnalisés (`tools.py`)

C'est ici que nous créons notre propre outil de recherche qui va interroger notre instance locale de SearxNG.

Créez un fichier `tools.py` dans votre dossier `~/autoblog` :

```python
# tools.py
import requests
import json
from langchain.tools import BaseTool

# --- Outil de recherche 100% local ---
class SearxNGSearchTool(BaseTool):
    name: str = "Local Search Engine"
    description: str = "Indispensable pour faire des recherches sur internet. Utilise une instance locale de SearxNG pour trouver des informations récentes ou des articles de blog."

    def _run(self, query: str) -> str:
        """Exécute une recherche sur l'instance locale de SearxNG."""
        try:
            searxng_url = "http://localhost:8080/"
            params = {
                'q': query,
                'format': 'json'
            }
            response = requests.get(searxng_url, params=params)
            response.raise_for_status()
            
            results = response.json().get('results', [])
            if not results:
                return "Aucun résultat trouvé pour cette recherche."

            # Formate les 3 premiers résultats pour que le LLM puisse les utiliser
            summary = "Voici les résultats de la recherche :\n"
            for i, res in enumerate(results[:3]):
                summary += f"Résultat {i+1}:\n"
                summary += f"  Titre: {res.get('title', 'N/A')}\n"
                summary += f"  URL: {res.get('url', 'N/A')}\n"
                summary += f"  Extrait: {res.get('content', 'N/A')}\n\n"
            return summary
        except requests.exceptions.RequestException as e:
            return f"Erreur de connexion au moteur de recherche local SearxNG : {e}"
        except Exception as e:
            return f"Une erreur est survenue lors de la recherche : {e}"

# --- Outils WordPress et Webhook (inchangés) ---

# Vos identifiants WordPress (à stocker dans des variables d'environnement !)
WP_URL = "https://VOTRE_SITE.COM/wp-json/wp/v2"
WP_USER = "VOTRE_USER"
WP_PASSWORD = "VOTRE_MOT_DE_PASSE_APPLICATION" # Important: utilisez un mot de passe d'application

class WordPressTool(BaseTool):
    name = "WordPress Tool"
    description = "Indispensable pour interagir avec un site WordPress. Permet de récupérer les tags existants et de publier de nouveaux articles."
    
    def _run(self, action: str, **kwargs):
        # ... (Le code de cette classe est identique à la réponse précédente) ...
        # ... (Copiez-collez le code de la classe WordPressTool ici) ...
        pass # Placeholder - remplacez par le vrai code

class WebhookTool(BaseTool):
    name = "Webhook Tool"
    description = "Utilisé pour envoyer des données (article, note, raison) à un endpoint spécifique via une requête POST."

    def _run(self, endpoint_url: str, data: dict):
        # ... (Le code de cette classe est identique à la réponse précédente) ...
        # ... (Copiez-collez le code de la classe WebhookTool ici) ...
        pass # Placeholder - remplacez par le vrai code

# Assurez-vous de copier le code complet pour WordPressTool et WebhookTool de la réponse précédente.
```

---

### Étape 4 : Le Code - Le Script Principal (`crew.py`)

Créez un fichier `crew.py` dans le même dossier. C'est le cerveau de l'opération.

```python
# crew.py
import os
import json
from crewai import Agent, Task, Crew, Process
from langchain_community.llms import Ollama
from crewai_tools import ScrapeWebsiteTool

# Importez VOS outils 100% locaux
from tools import SearxNGSearchTool, WordPressTool, WebhookTool

# --- Configuration du LLM Local ---
llm = Ollama(
    model="llama3:8b",
    base_url="http://localhost:11434"
)

# --- Initialisation des Outils ---
# On utilise notre outil maison au lieu de SerperDevTool
search_tool = SearxNGSearchTool() 
scrape_tool = ScrapeWebsiteTool()
wp_tool = WordPressTool()
webhook_tool = WebhookTool()

# --- Définition des Agents ---
# Les prompts sont les mêmes, mais on précise les outils locaux

news_crawler = Agent(
    role="Veilleur d'Actualités Tech utilisant des outils locaux",
    goal="Identifier un article pertinent et récent en utilisant le moteur de recherche interne.",
    backstory="Expert en veille, tu te fies uniquement aux outils fournis pour explorer le web. Tu es méticuleux et tu sais extraire l'URL la plus prometteuse des résultats de recherche.",
    tools=[search_tool, scrape_tool], # Outils 100% locaux
    llm=llm,
    verbose=True
)

# Les autres agents (strategic_writer, creative_editor, qa_judge) sont identiques à la réponse précédente.
# Ils utilisent déjà le `llm` local.
strategic_writer = Agent(...)
creative_editor = Agent(...)
qa_judge = Agent(...)

# Assurez-vous de copier le code de définition de ces 3 agents ici.
# Le `qa_judge` doit avoir un prompt très strict pour le format JSON.

# --- Définition des Tâches ---
# Les tâches sont identiques. On s'assure qu'elles utilisent les bons agents.
task_find_article = Task(...)
task_write_and_tag = Task(...)
task_enhance = Task(...)
task_judge = Task(...) # Le prompt de cette tâche est crucial pour la sortie JSON

# Assurez-vous de copier le code de définition des 4 tâches ici.

# --- Création et Lancement du Crew ---
publishing_crew = Crew(
    agents=[news_crawler, strategic_writer, creative_editor, qa_judge],
    tasks=[task_find_article, task_write_and_tag, task_enhance, task_judge],
    process=Process.sequential,
    verbose=2
)

# --- Logique de Décision Finale (Chef d'Orchestre) ---
if __name__ == '__main__':
    print("🚀 Lancement de l'équipe de rédaction IA (Mode 100% LOCAL)...")
    print("🕒 Soyez patient, le processus complet sur CPU est lent.")
    
    result_str = publishing_crew.kickoff()
    
    print("\n✅ Crew a terminé son travail. Analyse du résultat...")
    
    # Nettoyage de la sortie du LLM local, qui peut être "bavard"
    try:
        if "```json" in result_str:
            result_json_str = result_str.split("```json")[1].split("```")[0].strip()
        elif "```" in result_str:
             result_json_str = result_str.split("```")[1].split("```")[0].strip()
        else:
            result_json_str = result_str

        result = json.loads(result_json_str)
        
        # Le reste de la logique de décision est identique à la réponse précédente...
        # Copiez-collez ici la partie `if score > 8:` etc.
        # ...
        
    except (json.JSONDecodeError, IndexError) as e:
        print(f"❌ Erreur: Le Juge n'a pas retourné un JSON valide ou le format est inattendu. Erreur: {e}")
        print("--- Résultat brut reçu du Crew ---")
        print(result_str)
        print("---------------------------------")
    except Exception as e:
        print(f"❌ Une erreur inattendue est survenue : {e}")

```
**N'oubliez pas de copier-coller les parties manquantes (`...`) depuis la réponse précédente.**

---

### Étape 5 : Lancement et Automatisation

1.  **Lancement manuel pour tester :**
    *   Assurez-vous que votre service `docker-compose` (SearxNG) et Ollama tournent.
    *   Activez votre environnement virtuel : `source venv/bin/activate`
    *   Lancez le script : `python crew.py`
    *   Observez la console et soyez patient.

2.  **Automatisation avec Cron :**
    *   Pour que votre bot travaille pour vous (par exemple, tous les jours à 8h), utilisez un cron job.
    *   Ouvrez l'éditeur de cron : `crontab -e`
    *   Ajoutez cette ligne (adaptez les chemins !) :
    ```bash
    # Exécute le script de blog automatique tous les jours à 8h00
    0 8 * * * /home/VOTRE_USER/autoblog/venv/bin/python /home/VOTRE_USER/autoblog/crew.py >> /home/VOTRE_USER/autoblog/cron.log 2>&1
    ```
    *   **Explication de la ligne :**
        *   `0 8 * * *` : S'exécute à 8h00, tous les jours.
        *   `/home/VOTRE_USER/autoblog/venv/bin/python` : Chemin **absolu** vers l'interpréteur Python de votre environnement virtuel. C'est crucial !
        *   `/home/VOTRE_USER/autoblog/crew.py` : Chemin **absolu** vers votre script.
        *   `>> ... cron.log 2>&1` : Redirige toute la sortie (normale et erreurs) vers un fichier de log pour que vous puissiez déboguer en cas de problème.

Vous avez maintenant un système de publication de blog entièrement autonome, privé et gratuit, qui tourne sur votre propre matériel. C'est le summum de l'automatisation "Do It Yourself" 
