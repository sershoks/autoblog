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



##corectif##

Parfait ! C'est une autre erreur de validation, mais cette fois-ci, elle nous guide vers une solution beaucoup plus moderne et robuste.

### **L'Explication Simple et le Plan d'Action**

1.  **Le Warning (`LangChainDeprecationWarning`) :** LangChain est en pleine évolution. Ils ont séparé la librairie principale en plus petits paquets. Le message nous dit simplement que la façon d'appeler `Ollama` est maintenant dans un paquet dédié, `langchain-ollama`. C'est une bonne pratique de corriger cela.

2.  **L'Erreur (`ValidationError`) :** C'est le vrai problème. L'erreur `Input should be a valid dictionary or instance of BaseTool` est trompeuse. Bien que nos classes héritent bien de `BaseTool`, les versions récentes de CrewAI/LangChain/Pydantic préfèrent une manière plus simple et plus moderne de définir des outils : **utiliser le décorateur `@tool`** au lieu de créer des classes entières.

**Le plan est donc de :**
1.  Mettre à jour nos installations `pip` pour inclure le nouveau paquet `langchain-ollama`.
2.  Modifier `crew.py` pour utiliser la nouvelle façon d'appeler Ollama.
3.  **Réécrire complètement `tools.py`** en transformant nos classes en simples fonctions Python décorées avec `@tool`. C'est plus propre, plus simple et c'est ce que le système attend maintenant.
4.  Modifier `crew.py` pour qu'il utilise ces nouvelles fonctions-outils.

---

### **Étape 1 : Mettre à Jour les Dépendances**

Arrêtez votre script. Dans votre terminal avec l'environnement virtuel activé, exécutez cette commande pour installer le nouveau paquet et vous assurer que tout est à jour :

```bash
pip install -U crewai crewai-tools langchain-community langchain-ollama python-dotenv requests beautifulsoup4
```

---

### **Étape 2 : Réécriture Complète des Fichiers de Code**

Voici les versions entièrement corrigées et modernisées de `tools.py` et `crew.py`.

#### **Fichier `tools.py` - VERSION MODERNE**

Remplacez **tout le contenu** de votre fichier `tools.py` par ce qui suit. Remarquez que nous n'avons plus de classes, juste des fonctions.

```python
# tools.py
import os
import requests
import json
from langchain.tools import tool
from dotenv import load_dotenv

# Charge les variables depuis votre fichier .env
load_dotenv()

@tool("Moteur de Recherche Local")
def searxng_search_tool(query: str) -> str:
    """Indispensable pour faire des recherches sur internet. Utilise une instance locale de SearxNG pour trouver des informations récentes ou des articles de blog."""
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

@tool("Outil de récupération des tags WordPress")
def get_wordpress_tags() -> dict:
    """Permet de récupérer tous les tags existants sur le site WordPress pour les comparer."""
    WP_URL = os.getenv("WP_URL")
    WP_USER = os.getenv("WP_USER")
    WP_PASSWORD = os.getenv("WP_APPLICATION_PASSWORD")
    auth = (WP_USER, WP_PASSWORD)
    try:
        response = requests.get(f"{WP_URL}/tags?per_page=100", auth=auth, timeout=10)
        response.raise_for_status()
        tags = response.json()
        return {tag['name'].lower(): tag['id'] for tag in tags}
    except requests.exceptions.RequestException as e:
        return f"Erreur lors de la récupération des tags: {e}"

@tool("Outil de publication WordPress")
def publish_wordpress_post(title: str, content: str, tags_ids: list) -> str:
    """Publie un nouvel article sur le site WordPress avec un titre, un contenu et une liste d'IDs de tags."""
    WP_URL = os.getenv("WP_URL")
    WP_USER = os.getenv("WP_USER")
    WP_PASSWORD = os.getenv("WP_APPLICATION_PASSWORD")
    auth = (WP_USER, WP_PASSWORD)
    headers = {'Content-Type': 'application/json'}
    post_data = {'title': title, 'content': content, 'status': 'publish', 'tags': tags_ids}
    try:
        response = requests.post(f"{WP_URL}/posts", auth=auth, headers=headers, json=post_data, timeout=15)
        response.raise_for_status()
        return f"Article '{title}' publié avec succès !"
    except requests.exceptions.RequestException as e:
        return f"Erreur lors de la publication de l'article: {e}"

@tool("Outil Webhook")
def send_to_webhook(data: dict) -> str:
    """Utilisé pour envoyer les données d'un article rejeté à un endpoint de révision."""
    webhook_url = os.getenv("REJECTION_WEBHOOK_URL")
    try:
        response = requests.post(webhook_url, json=data, timeout=10)
        response.raise_for_status()
        return "Données envoyées au webhook de révision avec succès."
    except requests.exceptions.RequestException as e:
        return f"Erreur lors de l'envoi au webhook: {e}"
```

---

#### **Fichier `crew.py` - VERSION CORRIGÉE ET MODERNE**

Remplacez **tout le contenu** de votre fichier `crew.py`. Les changements sont importants : nous importons les nouvelles fonctions, nous les passons différemment aux agents, et nous les appelons directement à la fin.

```python
# crew.py
import os
import json
from crewai import Agent, Task, Crew, Process
from langchain_ollama import OllamaLLM # ### CHANGEMENT ### Correction du warning
from crewai_tools import ScrapeWebsiteTool
from dotenv import load_dotenv

# ### CHANGEMENT ### Importation des nouvelles fonctions-outils
from tools import searxng_search_tool, get_wordpress_tags, publish_wordpress_post, send_to_webhook

# Charge toutes les variables du fichier .env
load_dotenv()

# --- 1. Configuration et Initialisation des Outils ---
llm = OllamaLLM(model=os.getenv("OLLAMA_MODEL"), base_url=os.getenv("OLLAMA_BASE_URL")) # ### CHANGEMENT ### Correction du warning
scrape_tool = ScrapeWebsiteTool()
# Les autres outils sont maintenant des fonctions, pas besoin de les initialiser ici.

# --- 2. Définition des Agents ---
news_crawler = Agent(
    role="Veilleur d'Actualités Tech utilisant des outils locaux",
    goal="Identifier un article pertinent et récent en français sur la tech en utilisant le moteur de recherche interne. Fournir le contenu de l'article le plus prometteur.",
    backstory="Expert en veille, tu te fies uniquement aux outils fournis pour explorer le web. Tu es méticuleux et tu sais extraire l'URL la plus prometteuse des résultats de recherche pour ensuite en scraper le contenu.",
    tools=[searxng_search_tool, scrape_tool], # ### CHANGEMENT ### On passe les fonctions directement
    llm=llm,
    verbose=True,
    allow_delegation=False,
)

strategic_writer = Agent(
    role="Rédacteur Stratégique et SEO",
    goal="Créer un premier jet d'article unique basé sur le contenu fourni. Proposer une liste de 2 à 5 tags pertinents en vérifiant avec l'outil WordPress s'ils existent déjà pour les réutiliser.",
    backstory="Tu écris pour être lu et bien classé. Ta spécialité est de transformer une information brute en un article structuré et d'identifier les bons mots-clés (tags).",
    tools=[get_wordpress_tags], # ### CHANGEMENT ### On passe la fonction de récupération des tags
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

# --- 3. Définition des Tâches (inchangées) ---
task_find_article = Task(
    description="Cherche une actualité tech marquante des dernières 48h en français. Analyse les résultats, choisis l'article le plus intéressant, et scrape son contenu complet.",
    expected_output="Le contenu textuel complet de l'article choisi, nettoyé de tout élément non pertinent.",
    agent=news_crawler,
)

task_write_and_tag = Task(
    description=(
        "1. Lis le contenu de l'article fourni. Rédige un nouvel article en Markdown, inspiré de ce contenu mais avec tes propres mots.\n"
        "2. Utilise l'outil de récupération des tags WordPress pour obtenir la liste des tags existants.\n"
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
            # ### CHANGEMENT ### On appelle directement la fonction outil
            existing_tags_map = get_wordpress_tags()
            tag_ids = [existing_tags_map.get(name.lower()) for name in tags_names if name.lower() in existing_tags_map]
            
            # ### CHANGEMENT ### On appelle directement la fonction outil
            publication_status = publish_wordpress_post(title=title, content=content, tags_ids=[tid for tid in tag_ids if tid is not None])
            print(publication_status)
        else:
            print("🔴 Décision : Rejet. Envoi des données au webhook de révision.")
            rejection_data = {"titre": title, "contenu": content, "tags": tags_names, "note": score, "raison": reason}
            # ### CHANGEMENT ### On appelle directement la fonction outil
            status = send_to_webhook(data=rejection_data)
            print(status)
            
    except (json.JSONDecodeError, IndexError) as e:
        print(f"❌ Erreur: Le Juge n'a pas retourné un JSON valide ou le format est inattendu. Erreur: {e}")
        print("--- Résultat brut reçu du Crew ---")
        print(result_str)
    except Exception as e:
        print(f"❌ Une erreur inattendue est survenue : {e}")
```

### **Marche à Suivre Finale**

1.  Exécutez la commande `pip install -U ...` d'en haut.
2.  Remplacez le contenu de `tools.py` par la nouvelle version.
3.  Remplacez le contenu de `crew.py` par la nouvelle version.
4.  Lancez le script : `python crew.py`.

Cette nouvelle architecture est beaucoup plus alignée avec les dernières versions des librairies et devrait résoudre vos problèmes de validation.

Cette nouvelle architecture est beaucoup plus alignée avec les dernières versions des librairies et devrait résoudre vos problèmes de validation.




## ENCORE
Vous avez raison de poster ce traceback. C'est le signe d'un cycle de débogage frustrant, et cette erreur nous donne la clé finale. Je vous présente mes excuses pour les itérations précédentes ; l'écosystème LangChain/CrewAI évolue si vite que ces problèmes de compatibilité de version sont fréquents.

**Analyse de l'Erreur Finale**

*   **Le Problème :** `Input should be a valid dictionary or instance of BaseTool`
*   **La Cause Profonde :** Nous avons fait un cycle complet.
    1.  Notre **première tentative** avec les classes était la bonne idée, mais mal implémentée (manque de type `str`).
    2.  Notre **deuxième tentative** avec le décorateur `@tool` était la manière "moderne" de LangChain, mais la version de `CrewAI` que vous utilisez ne l'accepte pas dans le constructeur de l'`Agent`. Il s'attend *spécifiquement* à une instance d'une classe héritant de `BaseTool`.

**La Solution Définitive et Robuste**

Nous allons revenir à l'approche par classes, mais en l'implémentant de la manière la plus correcte et la plus moderne, qui est compatible avec Pydantic v2 et qui résoudra ce problème une bonne fois pour toutes. Cela implique de définir explicitement le "schéma" des arguments pour chaque outil.

C'est un peu plus verbeux, mais c'est la méthode la plus stable.

---

### **Fichier `tools.py` - LA VERSION FINALE ET ROBUSTE**

Remplacez **tout le contenu** de votre fichier `tools.py`. Cette version est la bonne.

```python
# tools.py
import os
import requests
import json
from typing import Type
from langchain.tools import BaseTool
from pydantic.v1 import BaseModel, Field # Important : Utiliser pydantic.v1 pour une meilleure compatibilité
from dotenv import load_dotenv

# Charge les variables depuis votre fichier .env
load_dotenv()

# --- Modèle pour l'entrée de l'outil de recherche ---
class SearchInput(BaseModel):
    query: str = Field(description="La chaîne de caractères à rechercher sur le web.")

class SearxNGSearchTool(BaseTool):
    name: str = "Moteur de Recherche Local"
    description: str = "Indispensable pour faire des recherches sur internet. Utilise une instance locale de SearxNG pour trouver des informations récentes."
    args_schema: Type[BaseModel] = SearchInput

    def _run(self, query: str) -> str:
        try:
            searxng_url = "http://localhost:8080/"
            params = {'q': query, 'format': 'json'}
            response = requests.get(searxng_url, params=params, timeout=10)
            response.raise_for_status()
            
            results = response.json().get('results', [])
            if not results: return "Aucun résultat trouvé."

            summary = "Voici les 3 premiers résultats :\n"
            for i, res in enumerate(results[:3]):
                summary += f"R {i+1}: Titre: {res.get('title', 'N/A')}, URL: {res.get('url', 'N/A')}, Extrait: {res.get('content', 'N/A')}\n"
            return summary
        except Exception as e:
            return f"Erreur lors de la recherche : {e}"

# --- Outils WordPress, maintenant séparés pour plus de clarté ---
class GetWordPressTagsTool(BaseTool):
    name: str = "Récupérateur de Tags WordPress"
    description: str = "Permet de récupérer tous les tags existants sur le site WordPress pour les comparer."
    
    def _run(self) -> dict:
        WP_URL, WP_USER, WP_PASSWORD = os.getenv("WP_URL"), os.getenv("WP_USER"), os.getenv("WP_APPLICATION_PASSWORD")
        try:
            response = requests.get(f"{WP_URL}/tags?per_page=100", auth=(WP_USER, WP_PASSWORD), timeout=10)
            response.raise_for_status()
            return {tag['name'].lower(): tag['id'] for tag in response.json()}
        except Exception as e:
            return f"Erreur lors de la récupération des tags : {e}"

class PublishPostInput(BaseModel):
    title: str = Field(description="Le titre de l'article à publier.")
    content: str = Field(description="Le contenu complet de l'article en format Markdown.")
    tags_ids: list = Field(description="Une liste d'IDs numériques des tags à associer à l'article.")

class PublishWordPressPostTool(BaseTool):
    name: str = "Outil de Publication WordPress"
    description: str = "Publie un nouvel article sur le site WordPress."
    args_schema: Type[BaseModel] = PublishPostInput

    def _run(self, title: str, content: str, tags_ids: list) -> str:
        WP_URL, WP_USER, WP_PASSWORD = os.getenv("WP_URL"), os.getenv("WP_USER"), os.getenv("WP_APPLICATION_PASSWORD")
        post_data = {'title': title, 'content': content, 'status': 'publish', 'tags': tags_ids}
        try:
            response = requests.post(f"{WP_URL}/posts", auth=(WP_USER, WP_PASSWORD), json=post_data, timeout=15)
            response.raise_for_status()
            return f"Article '{title}' publié avec succès !"
        except Exception as e:
            return f"Erreur lors de la publication : {e}"

# --- Outil Webhook ---
class WebhookInput(BaseModel):
    data: dict = Field(description="Un dictionnaire Python contenant les informations de l'article rejeté.")

class SendToWebhookTool(BaseTool):
    name: str = "Outil d'Envoi Webhook"
    description: str = "Utilisé pour envoyer les données d'un article rejeté à un endpoint de révision."
    args_schema: Type[BaseModel] = WebhookInput

    def _run(self, data: dict) -> str:
        try:
            response = requests.post(os.getenv("REJECTION_WEBHOOK_URL"), json=data, timeout=10)
            response.raise_for_status()
            return "Données envoyées au webhook avec succès."
        except Exception as e:
            return f"Erreur lors de l'envoi au webhook : {e}"
```

---

### **Fichier `crew.py` - LA VERSION CORRIGÉE POUR UTILISER LES BONS OUTILS**

Remplacez **tout le contenu** de votre fichier `crew.py`.

```python
# crew.py
import os
import json
from crewai import Agent, Task, Crew, Process
from langchain_ollama import OllamaLLM
from crewai_tools import ScrapeWebsiteTool
from dotenv import load_dotenv

# ### CHANGEMENT ### Importation des nouvelles classes d'outils
from tools import SearxNGSearchTool, GetWordPressTagsTool, PublishWordPressPostTool, SendToWebhookTool

load_dotenv()

# --- 1. Configuration et Initialisation des Outils ---
llm = OllamaLLM(model=os.getenv("OLLAMA_MODEL"), base_url=os.getenv("OLLAMA_BASE_URL"))
# ### CHANGEMENT ### On instancie chaque classe d'outil
scrape_tool = ScrapeWebsiteTool()
search_tool = SearxNGSearchTool()
get_tags_tool = GetWordPressTagsTool()
publish_post_tool = PublishWordPressPostTool()
webhook_tool = SendToWebhookTool()

# --- 2. Définition des Agents ---
news_crawler = Agent(
    role="Veilleur d'Actualités Tech",
    goal="Identifier un article pertinent et récent en français sur la tech en utilisant le moteur de recherche interne, puis scraper son contenu.",
    backstory="Expert en veille, tu utilises les outils fournis pour explorer le web, choisir l'article le plus prometteur et en extraire le contenu.",
    tools=[search_tool, scrape_tool], # ### CHANGEMENT ### On passe les instances des outils
    llm=llm,
    verbose=True,
)

strategic_writer = Agent(
    role="Rédacteur Stratégique et SEO",
    goal="Créer un premier jet d'article unique et proposer une liste de tags pertinents en utilisant l'outil de récupération de tags WordPress.",
    backstory="Tu transformes une information brute en un article structuré et identifies les bons mots-clés (tags).",
    tools=[get_tags_tool], # ### CHANGEMENT ### On passe l'instance de l'outil
    llm=llm,
    verbose=True,
)

# ... Les agents creative_editor et qa_judge n'ont pas d'outils, ils restent donc inchangés ...
creative_editor = Agent(
    role="Éditeur Créatif",
    goal="Prendre un article, le sublimer. Reformuler, enrichir avec des exemples, améliorer la fluidité et le rendre vraiment unique et agréable à lire.",
    backstory="Les mots sont ta matière première. Tu transformes un texte informatif en une histoire captivante.",
    llm=llm,
    verbose=True,
)

qa_judge = Agent(
    role="Juge Qualité Impitoyable",
    goal="Évaluer l'article final de manière objective. Attribuer une note de 1 à 10 et fournir une critique constructive. Le résultat DOIT être un JSON unique et valide.",
    backstory="Gardien de la qualité, ton jugement est juste et précis. Tu formates ta sortie rigoureusement pour la machine.",
    llm=llm,
    verbose=True,
)


# --- 3. Définition des Tâches (inchangées) ---
task_find_article = Task(...) # Le code des tâches ne change pas
task_write_and_tag = Task(...)
task_enhance = Task(...)
task_judge = Task(...)

# --- 4. Création et Lancement du Crew (inchangés) ---
publishing_crew = Crew(...)

# --- 5. Exécution et Logique de Décision ---
if __name__ == '__main__':
    print("🚀 Lancement de l'équipe de rédaction IA (Mode 100% LOCAL)...")
    result_str = publishing_crew.kickoff()
    print("\n✅ Crew a terminé. Analyse du résultat...")

    try:
        # ... Le code de parsing du JSON reste le même ...
        json_str = result_str
        if "```json" in json_str:
            json_str = json_str.split("```json")[1].split("```")[0].strip()
        elif "```" in json_str:
            json_str = json_str.split("```")[1].split("```")[0].strip()
        result = json.loads(json_str)

        score, reason, title, content, tags_names = (
            result.get('score', 0),
            result.get('reason', 'N/A'),
            result.get('final_title', 'N/A'),
            result.get('final_content', ''),
            result.get('final_tags', [])
        )
        print(f"--- Verdict du Juge ---\nNote : {score}/10\nRaison : {reason}\n-----------------------")

        if score >= 8:
            print("🟢 Décision : Publication sur WordPress.")
            # ### CHANGEMENT ### On appelle la méthode _run() de l'instance de l'outil
            existing_tags_map = get_tags_tool._run()
            tag_ids = [existing_tags_map.get(name.lower()) for name in tags_names if isinstance(existing_tags_map, dict) and name.lower() in existing_tags_map]
            
            # ### CHANGEMENT ### On appelle la méthode _run() de l'instance de l'outil
            publication_status = publish_post_tool._run(title=title, content=content, tags_ids=tag_ids)
            print(publication_status)
        else:
            print("🔴 Décision : Rejet. Envoi au webhook.")
            rejection_data = {"titre": title, "contenu": content, "tags": tags_names, "note": score, "raison": reason}
            # ### CHANGEMENT ### On appelle la méthode _run() de l'instance de l'outil
            status = webhook_tool._run(data=rejection_data)
            print(status)
            
    except Exception as e:
        print(f"❌ Erreur lors de l'analyse ou de l'exécution finale : {e}")
        print("--- Résultat brut reçu du Crew ---\n" + result_str)

```
**N.B. :** J'ai laissé les définitions de Tâches et du Crew en `...` car elles ne changent pas. Copiez-collez simplement les vôtres depuis votre fichier existant pour compléter ces sections.

### **Action Finale**

1.  Remplacez `tools.py` par le nouveau code.
2.  Remplacez `crew.py` par le nouveau code (en complétant les sections `...`).
3.  Lancez le script `python crew.py`.

Cette fois, la structure des outils correspond exactement à ce que le constructeur de `Agent` attend. L'erreur de validation Pydantic devrait être résolue.


