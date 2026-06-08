# ErrorFixer - Počítačová Databáze Chyb & Diagnostika

Tento projekt je komplexní databáze počítačových chyb. Slouží jako diagnostický nástroj všechny.

---

## 1. Úvod
* **Název projektu:** ErrorFixer
* **Téma:** Vyhledávání a diagnostika hardwarových a softwarových chyb PC, návody krok za krokem.
* **Živý web (GitHub Pages):** (https://matyasrada24.github.io/Web_Prace/) (web je propojen s vlastní doménou pomocí souboru `CNAME.txt` od hostingu Namecheap).

---

## 2. Použité technologie
* **Frontend:** HTML5, CSS3, JavaScript (ES6+)
* **Nástroje pro sestavení a optimalizaci:**
  * **Terser (v5.48.0):** Pokročilá minifikace, optimalizace a kompilace JavaScriptu.
  * **Python:** Automatizační, optimalizační a vyhodnocovací skripty.
* **IDE (Vývojové prostředí):** Visual Studio Code.

---

## 3. Adresářová struktura
Rozložení klíčových souborů a složek v projektu:

```text
c:\Users\PC\Desktop\Web\Error\
├── .git/                        # Git repozitář
├── admin/                       # Administrační rozhraní pro správu chyb
├── error/                       # Generované stránky detailů chyb (390+ složek)
│   ├── 0x00000024/index.html    # Příklad detailní stránky pro chybu NTFS_FILE_SYSTEM
│   └── ...
├── guides/                      # Diagnostické a opravné příručky (PC Build, Overheating atd.)
├── .htaccess                    # Konfigurace Apache serveru (CSP, cache, redirekce)
├── app.js                       # Minifikovaný produkční JavaScript (Terser)
├── style.css                    # Minifikovaný základní produkční CSS styl
├── detail-pages.css             # Minifikovaný produkční CSS styl pro detailní stránky
├── errors-db.js                 # Databáze chyb serializovaná do JSON řetězce (optimalizace V8)
├── index.html                   # Hlavní domovská stránka (interaktivní vyhledávač a průvodce)
├── sitemap.html                 # HTML mapa stránek
├── sitemap.xml                  # XML mapa stránek pro vyhledávače
├── CNAME.txt                    # Vlastní CNAME záznam pro GitHub Pages
```

---

### 4. Líné načítání tabulky chyb (Lazy Loading)
* **Teoretický popis:** Načítání a vykreslování 15 řádků HTML tabulky s ikonami a event listenery při startu blokuje hlavní vlákno mobilního CPU. Pomocí `IntersectionObserver` se odloží vykreslování řádků tabulky (funkce `L()`) na chvíli, kdy uživatel dosroluje do vzdálenosti 300px od sekce `#errors-section`. Pokud uživatel ihned použije vyhledávání nebo filtry, tabulka se okamžitě vykreslí.
* **Kódový výřez (app.js):**
  ```javascript
  // Odložení renderingu tabulky chyb na domovské stránce
  C(); // První filtrování dat (L() se přeskočí díky initialLoad)
  initialLoad = false;
  
  const errorsSection = document.getElementById("errors-section");
  if (errorsSection) {
      const observer = new IntersectionObserver((entries) => {
          entries.forEach(entry => {
              if (entry.isIntersecting) {
                  L(); // Spuštění renderingu tabulky chyb při srolování
                  observer.disconnect();
              }
          });
      }, { rootMargin: "300px" }); // Začne vykreslovat 300px před zobrazením na displeji
      observer.observe(errorsSection);
  }
  ```

### Rozdělení CSS stylů (CSS Splitting)
* **Teoretický popis:** Původní `style.css` (69KB) obsahoval veškerá pravidla jak pro domovskou stránku, tak pro všech 390+ stránek chyb a příruček. CSS parser mobilního prohlížeče musel analyzovat stovky pravidel, které na homepage nebyly použity. Rozdělil jsem CSS na základní `style.css` (40.8KB) a nový `detail-pages.css` (7.1KB). Tím klesla velikost CSS stahovaného na homepage, což urychlilo výpočty.
* **Kódový výřez (index.html vs. detailní stránky):**
  ```html
  <!-- index.html (Homepage načítá pouze základní styly) -->
  <link rel="preload" href="/style.css?v=1.0.9" as="style" onload="this.onload=null;this.rel='stylesheet'">
  
  <!-- error/*/index.html (Detailní stránky načítají oba soubory) -->
  <link rel="stylesheet" href="../../style.css?v=1.0.9"/>
  <link rel="stylesheet" href="../../detail-pages.css?v=1.0.9"/>
  ```

### Hromadná aktualizace stránek
* **Teoretický popis:** Provedení změn v odkazování na CSS ve všech 408 vygenerovaných HTML stránkách ručně by bylo neefektivní a náchylné k chybám. Vytvořil jsme Python skript, který prošel adresářovou strukturu, automaticky detekoval relativní hloubku cesty k CSS a vložil odkaz na `detail-pages.css` hned pod stávající odkaz pro `style.css` včetně správné verze.
* **Kódový výřez (update_all_pages_css.py):**
  ```python
  # Regex vyhledání a vložení nového detail-pages.css
  stylesheet_pattern = re.compile(
      r'(<link rel="stylesheet" href="([^"]*?)style\.css(?:\?v=[\d\.]+)?"\s*/?>)'
  )
  
  if stylesheet_pattern.search(content):
      def repl_sheet(m):
          orig = m.group(0)
          pref = m.group(2) # Relativní prefix např. ../../
          added = f'\n<link rel="stylesheet" href="{pref}detail-pages.css{version}"/>'
          return orig + added
      new_content, count = stylesheet_pattern.subn(repl_sheet, content)
  ```

### Zrychlení parsování databáze (JSON.parse Trick)
* **Teoretický popis:** Databáze `errors-db.js` má přes 319KB. Prohlížeče (zejména engine V8 v Google Chrome a mobilních zařízeních) parsují velké řetězce předané přes `JSON.parse('...')` až 10x rychleji než standardní JS objekty a pole. Tím se ušetří významný čas v sekci Script Evaluation při inicializaci aplikace.
* **Kódový výřez (errors-db.js):**
  ```javascript
  // Data jsou zabalena do jednoho řetězce a parsována přímo V8 enginem
  const ERRORS = JSON.parse('[{"id": "CPU-001", "code": "WHEA_UNCORRECTABLE_ERROR", ...}]');
  ```

### 4.5 Odstranění Umami Analytics & CSP Cleanup
* **Teoretický popis:** Google Lighthouse audit vykazoval chyby v sekci Best Practices z důvodu používání zastaralého persistentního Storage API ze strany měřicího skriptu Umami Analytics. Kompletním odstraněním injektáže Umami skriptu v klientském JS a vyčištěním nepoužívaných domén z Content Security Policy v `.htaccess` jsem vyřešili všechna Lighthouse varování.
* **Kódový výřez (.htaccess):**
  ```apache
  # Očištěná hlavička CSP (bez domén umami.is a api-gateway.umami.dev)
  Header set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://pagead2.googlesyndication.com; ..."
  ```

### Automatizace sestavení (minify_assets.py)
* **Teoretický popis:** Pro spolehlivou minifikaci a testování integrity souborů před nasazením na produkční hosting byl upraven sestavovací skript. Ten automaticky obnovuje čisté CSS ze zdrojových `.src.css` souborů, minifikuje je, spouští kompilaci JS přes Terser a sestavuje finální balíček `upload.zip`.
* **Kódový výřez (minify_assets.py):**
  ```python
  def run_optimization():
      rebuild_css() # Obnoví style.css a detail-pages.css ze zdrojů
      # Minifikuje style.css
      # Minifikuje detail-pages.css
      compile_js()  # Spustí npx terser app.src.js -o app.js ...
      optimize_html("index.html")
      create_zip()  # Zabalí minifikovaný build do upload.zip
  ```

---

## 5. AI Deník
Seznam zajímavých promptů a přínosů umělé inteligence během vývoje a optimalizace projektu:

* **Zajímavé prompty:**
  * *"Jak mohu snížit čas blokování hlavního vlákna (TBT) na mobilním telefonu pro statický web, kde načítám velkou databázi chyb v JS?"*
  * *"Navrhni způsob, jak rozdělit 69KB CSS na kritické pro homepage a odložené pro detailní stránky bez narušení funkčnosti."*
  * *"Napiš Python skript, který projde 400+ HTML souborů v podadresářích, najde relativní cestu k style.css a pod ni spolehlivě vloží odkaz na detail-pages.css."*
  * *"Jak můžu zrychlit načítání obřího pole objektů v JS na mobilním Chrome? Slyšel jsem o triku s JSON.parse."*

* **Přínos AI do projektu:**
  * **Analýza slabých míst:** Rychlá lokalizace problémů s výkonem na základě reportu Google Lighthouse.
  * **Automatizace:** Tvorba robustních Python skriptů pro hromadnou editaci stovek souborů bez nutnosti manuální práce.
  * **Pokročilé techniky:** Implementace `IntersectionObserver` pro lazy-rendering a `JSON.parse` optimalizace pro maximální zrychlení parsování skriptů na telefonech.

---

## 6. Instalace a spuštění

### Lokální zprovoznění
Pro lokální vývoj a testování doporučuji použít extension **Live Server** ve VS Code:
1. Otevřete složku projektu ve VS Code.
2. Nainstalujte extension **Live Server** (od Ritwick Dey).
3. Klikněte pravým tlačítkem na soubor `index.html` a zvolte **Open with Live Server**.
4. Web se otevře na adrese `http://127.0.0.1:5500/`.

---

## 7. Galerie
Náhledy rozhraní a klíčových funkcí webové aplikace:

### Domovská stránka (Desktop)
![image alt](https://github.com/MatyasRada24/Web_Prace/blob/506cd2373a251cbdefc58afef68ec6629a570aa5/Images/zacatek.PNG)

### Domovská stránka (Telefon)
![image alt](https://github.com/MatyasRada24/Web_Prace/blob/506cd2373a251cbdefc58afef68ec6629a570aa5/Images/zacatek_telefon.PNG)

### Mobilní zobrazení a optimalizované detaily chyb
Aplikace je plně responzivní. Mobilní verze nepoužívá náročné canvas částice na pozadí (`#particleCanvas { display: none; }` pro displeje pod 768px), což výrazně šetří výkon mobilního procesoru a prodlužuje výdrž baterie zařízení. Tabulka chyb se načítá plynule při scrollování a vyhledávání nabízí okamžitý našeptávač.
