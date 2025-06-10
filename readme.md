Absolument. Vous voulez la recette complète, sans les trous, prête à l'emploi. C'est parti.

Nous allons construire un système 100% local, fonctionnant sur CPU, avec tous les scripts complets et un fichier de configuration dédié.

---

### **Aperçu du Projet Final**

*   **Dossier du Projet :** Tout sera dans un dossier (ex: `~/autoblog`).
*   **Configuration :** Un seul fichier `.env` pour toutes vos variables (identifiants, URLs).
*   **Infrastructure :** Ollama pour l'IA, SearxNG pour la recherche, le tout tournant sur votre serveur.
*   **Code :** Deux fichiers Python complets : `tools.py` pour les capacités de vos agents, et `crew.py` pour orchestrer le tout.

---

### **Étape 1 : Le Fichier de Configuration (`.env`)**

C'est la première et unique chose que vous aurez à éditer. Créez un fichier nommé `.env` dans le dossier de votre projet (`~/autoblog/.env`).

**Contenu à copier-coller dans votre fichier `.env` :**

```env
# --- Configuration WordPress ---
# L'URL de votre site WordPress avec le chemin vers l'API REST
WP_URL="https://VOTRE_SITE.com/wp-json/wp/v2"

# Votre nom d'utilisateur WordPress
WP_USER="VOTRE_NOM_UTILISATEUR"

# IMPORTANT: N'utilisez PAS votre mot de passe principal.
# Allez dans votre profil WordPress -> Mots de passe d'application -> Créez-en un nouveau.
WP_APPLICATION_PASSWORD="VOTRE_MOT_DE_PASSE_APPLICATION"

# --- Configuration du Webhook de Rejet ---
# L'URL où envoyer les articles qui n'ont pas une note suffisante
# Vous pouvez utiliser un service comme n8n, Make, ou un script personnalisé pour créer ce webhook.
REJECTION_WEBHOOK_URL="https://VOTRE_ENDPOINT_WEBHOOK.com/rejet"

# --- Configuration du Modèle IA Local ---
# Le nom du modèle que vous utilisez dans Ollama (ici, llama3:8b)
OLLAMA_MODEL="llama3:8b"

# L'adresse de votre serveur Ollama (laisser par défaut si sur la même machine)
OLLAMA_BASE_URL="http://localhost:11434"
```

---

### **Étape 2 : Mise en Place de l'Infrastructure Locale**

Ces commandes sont à exécuter sur votre serveur.

**1. Installer Docker & Docker Compose :**
Si ce n'est pas déjà fait, suivez les guides officiels pour votre distribution Linux. C'est un prérequis.

**2. Installer Ollama :**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**3. Télécharger votre modèle LLM :**
```bash
# Télécharge et prépare le modèle Llama 3 8B
ollama pull llama3:8b
```
Ollama tournera ensuite en tâche de fond, prêt à l'emploi.

**4. Mettre en place le moteur de recherche local SearxNG :**
Dans votre dossier de projet (`~/autoblog`), créez le fichier `docker-compose.yml` :
```yaml
# docker-compose.yml
version: '3.8'

services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - ./searxng_data:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://localhost:8080/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
```
Lancez-le avec `docker-compose up -d`. Votre moteur de recherche privé sera accessible sur `http://localhost:8080`.

---

### **Étape 3 : Préparation de l'Environnement Python**

```bash
# Placez-vous dans votre dossier de projet
cd ~/autoblog

# Créez un environnement virtuel
python3 -m venv venv

# Activez-le
source venv/bin/activate

# Installez toutes les librairies nécessaires
pip install crewai requests beautifulsoup4 langchain-community python-dotenv
```

---

### **Étape 4 : Le Code Source Complet**

Voici les deux fichiers Python, complets et prêts à être utilisés.

#### **Fichier 1 : `tools.py`**

Ce fichier définit les "super-pouvoirs" de vos agents : chercher sur le web local, interagir avec WordPress, etc.

```python
import os
import requests
import json
from langchain.tools import BaseTool
from dotenv import load_dotenv

# Charge les variables depuis votre fichier .env
load_dotenv()

class SearxNGSearchTool(BaseTool):
    # CORRECTION : Ajout de l'annotation de type ': str'
    name: str = "Moteur de Recherche Local"
    description: str = "Indispensable pour faire des recherches sur internet. Utilise une instance locale de SearxNG pour trouver des informations récentes ou des articles de blog."

    def _run(self, query: str) -> str:
        """Exécute une recherche sur l'instance locale de SearxNG."""
        try:
            searxng_url = "http://localhost:8080/"
            params = {'q': query, 'format': 'json'}
            response = requests.get(searxng_url, params=params, timeout=10)
            response.raise_for_status()
            
            results = response.json().get('results', [])
            if not results:
                return "Aucun résultat trouvé pour cette recherche."

            summary = "Voici les 3 premiers résultats de la recherche :\n"
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

class WordPressTool(BaseTool):
    # CORRECTION : Ajout de l'annotation de type ': str'
    name: str = "Outil WordPress"
    description: str = "Indispensable pour interagir avec un site WordPress. Permet de récupérer les tags existants et de publier de nouveaux articles."

    def _run(self, action: str, title: str = None, content: str = None, tags_ids: list = None):
        """Exécute une action sur WordPress."""
        WP_URL = os.getenv("WP_URL")
        WP_USER = os.getenv("WP_USER")
        WP_PASSWORD = os.getenv("WP_APPLICATION_PASSWORD")
        auth = (WP_USER, WP_PASSWORD)
        headers = {'Content-Type': 'application/json'}

        if action == "get_existing_tags":
            try:
                response = requests.get(f"{WP_URL}/tags?per_page=100", auth=auth, timeout=10)
                response.raise_for_status()
                tags = response.json()
                return {tag['name'].lower(): tag['id'] for tag in tags}
            except requests.exceptions.RequestException as e:
                return f"Erreur lors de la récupération des tags: {e}"
        
        elif action == "publish_post":
            if not all([title, content, tags_ids is not None]):
                return "Erreur: Le titre, le contenu et la liste des IDs de tags sont requis pour publier."
            
            post_data = {'title': title, 'content': content, 'status': 'publish', 'tags': tags_ids}
            try:
                response = requests.post(f"{WP_URL}/posts", auth=auth, headers=headers, json=post_data, timeout=15)
                response.raise_for_status()
                return f"Article '{title}' publié avec succès !"
            except requests.exceptions.RequestException as e:
                return f"Erreur lors de la publication de l'article: {e}"
        
        else:
            return "Action non reconnue. Utilisez 'get_existing_tags' ou 'publish_post'."

class WebhookTool(BaseTool):
    # CORRECTION : Ajout de l'annotation de type ': str'
    name: str = "Outil Webhook"
    description: str = "Utilisé pour envoyer des données (article, note, raison) à un endpoint spécifique via une requête POST."

    def _run(self, data: dict):
        """Envoie les données au webhook de rejet."""
        webhook_url = os.getenv("REJECTION_WEBHOOK_URL")
        try:
            response = requests.post(webhook_url, json=data, timeout=10)
            response.raise_for_status()
            return "Données envoyées au webhook de révision avec succès."
        except requests.exceptions.RequestException as e:
            return f"Erreur lors de l'envoi au webhook: {e}"

```

#### **Fichier 2 : `crew.py`**

Ce fichier est le chef d'orchestre. Il définit les agents, les tâches, et exécute le travail.

```python
# crew.py
import os
import json
from crewai import Agent, Task, Crew, Process
from langchain_community.llms import Ollama
from crewai_tools import ScrapeWebsiteTool
from dotenv import load_dotenv

from tools import SearxNGSearchTool, WordPressTool, WebhookTool

# Charge toutes les variables du fichier .env
load_dotenv()

# --- 1. Configuration et Initialisation des Outils ---
llm = Ollama(model=os.getenv("OLLAMA_MODEL"), base_url=os.getenv("OLLAMA_BASE_URL"))
search_tool = SearxNGSearchTool()
scrape_tool = ScrapeWebsiteTool()
wp_tool = WordPressTool()
webhook_tool = WebhookTool()

# --- 2. Définition des Agents ---
news_crawler = Agent(
    role="Veilleur d'Actualités Tech utilisant des outils locaux",
    goal="Identifier un article pertinent et récent en français sur la tech en utilisant le moteur de recherche interne. Fournir le contenu de l'article le plus prometteur.",
    backstory="Expert en veille, tu te fies uniquement aux outils fournis pour explorer le web. Tu es méticuleux et tu sais extraire l'URL la plus prometteuse des résultats de recherche pour ensuite en scraper le contenu.",
    tools=[search_tool, scrape_tool],
    llm=llm,
    verbose=True,
    allow_delegation=False,
)

strategic_writer = Agent(
    role="Rédacteur Stratégique et SEO",
    goal="Créer un premier jet d'article unique basé sur le contenu fourni. Proposer une liste de 2 à 5 tags pertinents en vérifiant avec l'outil WordPress s'ils existent déjà pour les réutiliser.",
    backstory="Tu écris pour être lu et bien classé. Ta spécialité est de transformer une information brute en un article structuré et d'identifier les bons mots-clés (tags).",
    tools=[wp_tool],
    llm=llm,
    verbose=True,
)

creative_editor = Agent(
    role="Éditeur Créatif",
    goal="Prendre un article, le sublimer. Reformuler, enrichir avec des exemples, améliorer la fluidité et le rendre vraiment unique et agréable à lire. Le ton doit être professionnel mais accessible.",
    backstory="Les mots sont ta matière première. Tu transformes un texte informatif en une histoire captivante sans jamais inventer d'informations.",
    llm=llm,
    verbose=True,
)

qa_judge = Agent(
    role="Juge Qualité Impitoyable",
    goal="Évaluer l'article final de manière objective. Attribuer une note de 1 à 10 et fournir une critique constructive. Le résultat DOIT être un JSON unique et valide.",
    backstory="Tu es le gardien de la qualité. Ton jugement est juste, basé sur la clarté, l'originalité et la structure. Tu dois formater ta sortie de manière extrêmement précise pour que la machine puisse la comprendre.",
    llm=llm,
    verbose=True,
)

# --- 3. Définition des Tâches ---
task_find_article = Task(
    description="Cherche une actualité tech marquante des dernières 48h en français. Analyse les résultats, choisis l'article le plus intéressant, et scrape son contenu complet.",
    expected_output="Le contenu textuel complet de l'article choisi, nettoyé de tout élément non pertinent.",
    agent=news_crawler,
)

task_write_and_tag = Task(
    description=(
        "1. Lis le contenu de l'article fourni. Rédige un nouvel article en Markdown, inspiré de ce contenu mais avec tes propres mots.\n"
        "2. Utilise l'outil WordPress pour obtenir la liste des tags existants.\n"
        "3. Propose une liste de 2 à 5 noms de tags pertinents pour l'article, en privilégiant les noms qui existent déjà."
    ),
    expected_output="Un objet Python contenant deux clés: 'article_draft' (le premier jet de l'article en Markdown) et 'suggested_tags' (une liste de noms de tags).",
    agent=strategic_writer,
    context=[task_find_article],
)

task_enhance = Task(
    description="Prends le premier jet de l'article et la liste de tags. Améliore l'article: rends-le plus engageant, ajoute de la profondeur, reformule les phrases pour un style unique. Ne modifie pas les tags suggérés.",
    expected_output="Un article finalisé en Markdown, prêt pour une évaluation de qualité, accompagné de la liste de tags non modifiée.",
    agent=creative_editor,
    context=[task_write_and_tag],
)

task_judge = Task(
    description="Analyse l'article finalisé. Donne une note de 1 à 10. Justifie ta note. Propose un titre final accrocheur. Formate ta réponse en un bloc de code JSON unique et valide, sans aucun texte avant ou après.",
    expected_output="""Un objet JSON valide contenant les clés suivantes :
    - "score": (integer) la note de 1 à 10.
    - "reason": (string) une explication détaillée de la note.
    - "final_title": (string) le titre final suggéré pour l'article.
    - "final_content": (string) le contenu complet de l'article en Markdown.
    - "final_tags": (list of strings) la liste des noms de tags.
    """,
    agent=qa_judge,
    context=[task_enhance],
)

# --- 4. Création et Lancement du Crew ---
publishing_crew = Crew(
    agents=[news_crawler, strategic_writer, creative_editor, qa_judge],
    tasks=[task_find_article, task_write_and_tag, task_enhance, task_judge],
    process=Process.sequential,
    verbose=2,
)

# --- 5. Exécution et Logique de Décision ---
if __name__ == '__main__':
    print("🚀 Lancement de l'équipe de rédaction IA (Mode 100% LOCAL)...")
    print("🕒 Soyez patient, le processus complet sur CPU est lent.")
    
    result_str = publishing_crew.kickoff()
    
    print("\n✅ Crew a terminé son travail. Analyse du résultat...")

    try:
        # Nettoyage de la sortie du LLM local, qui peut être "bavard"
        json_str = result_str
        if "```json" in json_str:
            json_str = json_str.split("```json")[1].split("```")[0].strip()
        elif "```" in json_str:
            json_str = json_str.split("```")[1].split("```")[0].strip()

        result = json.loads(json_str)
        
        score = result.get('score', 0)
        reason = result.get('reason', 'Raison non fournie.')
        title = result.get('final_title', 'Titre non fourni.')
        content = result.get('final_content', 'Contenu non fourni.')
        tags_names = result.get('final_tags', [])

        print(f"--- Verdict du Juge ---")
        print(f"Note : {score}/10")
        print(f"Raison : {reason}")
        print("-----------------------")

        if score >= 8:
            print("🟢 Décision : Publication sur WordPress.")
            existing_tags_map = wp_tool._run(action="get_existing_tags")
            tag_ids = [existing_tags_map.get(name.lower()) for name in tags_names if name.lower() in existing_tags_map]
            
            publication_status = wp_tool._run(action="publish_post", title=title, content=content, tags_ids=[tid for tid in tag_ids if tid is not None])
            print(publication_status)
        else:
            print("🔴 Décision : Rejet. Envoi des données au webhook de révision.")
            rejection_data = {"titre": title, "contenu": content, "tags": tags_names, "note": score, "raison": reason}
            status = webhook_tool._run(data=rejection_data)
            print(status)
            
    except (json.JSONDecodeError, IndexError) as e:
        print(f"❌ Erreur: Le Juge n'a pas retourné un JSON valide ou le format est inattendu. Erreur: {e}")
        print("--- Résultat brut reçu du Crew ---")
        print(result_str)
    except Exception as e:
        print(f"❌ Une erreur inattendue est survenue : {e}")

```

---

### **Étape 5 : Lancement et Automatisation**

1.  **Lancement manuel (pour tester) :**
    *   Assurez-vous que Docker (avec SearxNG) et Ollama tournent.
    *   Dans votre terminal, activez l'environnement : `source venv/bin/activate`
    *   Lancez le script : `python crew.py`
    *   Observez la magie opérer... lentement.

2.  **Automatisation (pour la production) :**
    *   Utilisez `cron` pour lancer le script automatiquement.
    *   Ouvrez l'éditeur de cron : `crontab -e`
    *   Ajoutez cette ligne en adaptant les chemins avec VOS chemins absolus :

    ```bash
    # Exécute le script de blog automatique tous les jours à 8h00 du matin
    0 8 * * * /home/VOTRE_USER/autoblog/venv/bin/python /home/VOTRE_USER/autoblog/crew.py >> /home/VOTRE_USER/autoblog/cron.log 2>&1
    ```
    Cela exécutera votre bot tous les matins et enregistrera sa sortie dans un fichier `cron.log` pour que vous puissiez vérifier que tout s'est bien passé.

Vous disposez maintenant d'une solution complète, documentée et prête à être déployée.
