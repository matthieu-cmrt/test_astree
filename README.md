# test_astree

```js
/* =====================================================================
   maj_membres v2 — page "Associés / Participants"
   Pour chaque personne du CSV : recherche dans la liste, clic sur le nom,
   clic sur "Modifier la fiche", activation des changements, mise à jour, enregistrement.
   Champs pris en charge (selon les colonnes présentes dans le CSV) :
   Genre, Date de naissance, Section de vote, Titulaire / suppléant. Email = vérification.
   À coller TEL QUEL dans la console (F12) sur la page des participants :
   une fenêtre s'ouvre ensuite pour coller le CSV.
   ===================================================================== */
(async () => {

  /* ---------- 1. LE CSV ----------
     Rien à modifier ici : au lancement, une fenêtre s'ouvre sur la page pour coller le CSV
     ou choisir le fichier. (Colonnes reconnues : Prénom;Nom;Email;Genre;Date de naissance;
     Section de vote;Titulaire / suppléant — seules les colonnes présentes sont traitées.) */
  function demanderCSV() {
    return new Promise(resolve => {
      document.getElementById('maj-csv-panel')?.remove();
      const box = document.createElement('div');
      box.id = 'maj-csv-panel';
      box.style.cssText = 'position:fixed;top:20px;right:20px;z-index:2147483647;width:min(560px,calc(100vw - 40px));' +
        'background:#fff;color:#1b2330;border:2px solid #2c4bb8;border-radius:10px;padding:16px;' +
        'box-shadow:0 10px 30px rgba(0,0,0,.25);font:14px system-ui,sans-serif;display:grid;gap:10px';
      box.innerHTML =
        '<div style="font-weight:600;font-size:16px">maj_membres v2 — données à traiter</div>' +
        '<div style="color:#5d6878">Colle le contenu du CSV (ligne d\'en-tête comprise) ou choisis le fichier.</div>' +
        '<textarea style="width:100%;height:220px;font:12px monospace;padding:8px;border:1px solid #ccd;border-radius:6px;box-sizing:border-box" ' +
        'placeholder="Prénom;Nom;Email;Genre;Date de naissance&#10;Camille;DURAND;camille.durand@exemple.fr;F;05/03/1972"></textarea>' +
        '<div style="display:flex;gap:8px;align-items:center;flex-wrap:wrap">' +
        '<label style="cursor:pointer;color:#2c4bb8;text-decoration:underline">Choisir un fichier .csv<input type="file" accept=".csv,.txt" style="display:none"></label>' +
        '<span class="info" style="color:#5d6878;flex:1"></span>' +
        '<button class="annuler" style="padding:6px 12px;border:1px solid #ccd;border-radius:6px;background:#fff;cursor:pointer">Annuler</button>' +
        '<button class="lancer" style="padding:6px 14px;border:0;border-radius:6px;background:#2c4bb8;color:#fff;font-weight:600;cursor:pointer">Lancer</button></div>';
      document.body.appendChild(box);
      const ta = box.querySelector('textarea'), info = box.querySelector('.info');
      const compter = () => { const n = ta.value.split(/\r?\n/).filter(l => l.trim()).length; info.textContent = n ? `${n} ligne(s), en-tête comprise` : ''; };
      ta.addEventListener('input', compter);
      box.querySelector('input[type=file]').addEventListener('change', async e => {
        const f = e.target.files[0]; if (!f) return;
        const buf = await f.arrayBuffer();
        let txt; try { txt = new TextDecoder('utf-8', { fatal: true }).decode(buf); } catch (x) { txt = new TextDecoder('windows-1252').decode(buf); }
        ta.value = txt.replace(/^﻿/, ''); compter();
      });
      box.querySelector('.annuler').onclick = () => { box.remove(); resolve(null); };
      box.querySelector('.lancer').onclick = () => {
        if (!ta.value.trim()) { info.textContent = '⚠️ Colle d’abord le CSV.'; return; }
        box.remove(); resolve(ta.value);
      };
      ta.focus();
    });
  }
  const CSV = (window.CSV_MAJ && String(window.CSV_MAJ).trim()) ? String(window.CSV_MAJ) : await demanderCSV();
  if (!CSV) { console.log('⏹️ Annulé.'); return; }

  /* ---------- 2. RÉGLAGES ---------- */
  const CONFIG = {
    DEPART: 1,             // n° de ligne du CSV où commencer (pour reprendre après un arrêt)
    LIMITE: 0,            // nb de personnes du CSV à traiter (0 = toutes, 1 = test sur une seule)
    DRY_RUN: false,        // true = remplit tout mais clique sur "Fermer" au lieu d'"Enregistrer"
    CORRIGER_GENRE: true,  // true = corrige le genre s'il ne correspond pas au CSV
    TOUTES_LES_LIGNES: true, // si plusieurs participants portent le même nom, les traiter tous (l'e-mail départage)
    DELAI: 500,            // pause (ms) entre les actions
    PAUSE_VISUELLE: 1500,  // temps (ms) pour voir les contours vert/orange avant d'enregistrer
  };

  /* ---------- Outils ---------- */
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const norm = s => (s || '').normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .toLowerCase().replace(/[-'’]/g, ' ').replace(/\s+/g, ' ').trim();
  // Clé insensible à l'ordre prénom/nom, aux accents et à la casse
  const cle = s => norm(s).split(' ').filter(Boolean).sort().join(' ');
  const visible = el => el && el.offsetParent !== null;

  async function attendre(fn, timeout = 10000, pas = 150) {
    const t0 = Date.now();
    while (Date.now() - t0 < timeout) {
      try { const r = fn(); if (r) return r; } catch (e) {}
      await sleep(pas);
    }
    return null;
  }

  // Trouve un champ via le texte de son <label> (les id sont générés dynamiquement)
  function champ(libelle, tag = 'input') {
    const cible = norm(libelle);
    for (const lab of document.querySelectorAll('label')) {
      if (norm(lab.textContent) !== cible) continue;
      const grp = lab.closest('.form-group, .form-group-default__custom') || lab.parentElement;
      const el = grp && grp.querySelector(tag);
      if (visible(el)) return el;
    }
    return null;
  }

  // Champ dont le label COMMENCE par un texte (ex. "Date de naissance", "DATE DE NAISSANCE / CRÉATION")
  function champCommencant(debut, tag = 'input') {
    const cible = norm(debut);
    for (const lab of document.querySelectorAll('label')) {
      if (!norm(lab.textContent).startsWith(cible)) continue;
      const grp = lab.closest('.form-group, .form-group-default__custom') || lab.parentElement;
      const el = grp && grp.querySelector(tag);
      if (visible(el)) return el;
    }
    return null;
  }

  // Champ E-mail de la fiche (label "E-mail" ou "Email", mais pas "E-mail CC")
  function champEmail() {
    for (const lab of document.querySelectorAll('label')) {
      const t = norm(lab.textContent).replace(/\s/g, '');
      if (!t.startsWith('email') || t.includes('cc')) continue;
      const grp = lab.closest('.form-group, .form-group-default__custom') || lab.parentElement;
      const el = grp && grp.querySelector('input');
      if (visible(el)) return el;
    }
    return null;
  }

  // Interrupteur "Activer les changements" de la fiche
  function activerModification() {
    const blocSwitch = [...document.querySelectorAll('div')]
      .find(div => visible(div) && norm(div.textContent) === 'activer les changements');
    const inter = blocSwitch?.parentElement.querySelector('.switch input[type=checkbox]');
    if (inter && !inter.checked) inter.click();
  }

  // Dates : tout format → AAAA-MM-JJ (ou null), et AAAA-MM-JJ → JJ/MM/AAAA
  const p2 = n => String(n).padStart(2, '0');
  function dateISO(v) {
    const s = String(v ?? '').trim();
    if (!s) return null;
    let m = s.match(/^(\d{4})-(\d{1,2})-(\d{1,2})/);
    if (m) return `${m[1]}-${p2(m[2])}-${p2(m[3])}`;
    m = s.match(/^(\d{1,2})[\/.\-](\d{1,2})[\/.\-](\d{2,4})$/);
    if (m) {
      let y = m[3];
      if (y.length === 2) y = (Number(y) <= new Date().getFullYear() % 100 ? '20' : '19') + y;
      if (Number(m[2]) > 12 || Number(m[1]) > 31) return null;
      return `${y}-${p2(m[2])}-${p2(m[1])}`;
    }
    if (/^\d{4,5}$/.test(s)) { // n° de série Excel
      const d = new Date(Math.round((Number(s) - 25569) * 86400000));
      return `${d.getUTCFullYear()}-${p2(d.getUTCMonth() + 1)}-${p2(d.getUTCDate())}`;
    }
    return null;
  }
  const dateFR = iso => { const [y, m, d] = iso.split('-'); return `${d}/${m}/${y}`; };

  // Saisie compatible Blazor (déclenche input + change)
  function saisir(el, val) {
    const proto = el instanceof HTMLSelectElement ? HTMLSelectElement.prototype : HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, val);
    el.dispatchEvent(new Event('input', { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  }

  // Check visuel : vert = déjà bon, orange = modifié par le script
  function marquer(el, dejaBon) {
    const c = dejaBon ? '#28a745' : '#fd7e14';
    el.style.setProperty('border', `2px solid ${c}`, 'important');
    el.style.setProperty('box-shadow', `0 0 0 2px ${c}55`, 'important');
  }

  // Remplit le champ seulement si la valeur diffère, et le colore. Renvoie true si déjà bon.
  function remplir(el, val) {
    const dejaBon = (el.value || '').trim().toLowerCase() === (val || '').trim().toLowerCase();
    if (!dejaBon) saisir(el, val);
    marquer(el, dejaBon);
    return dejaBon;
  }

  // Bouton "Enregistrer" : <button>, <a class="btn"> ou <input type=submit>, le plus récent (= la fiche au premier plan)
  const boutonEnregistrer = () => [...document.querySelectorAll('button, a.btn, input[type=submit]')]
    .filter(b => visible(b) && norm(b.textContent || b.value).startsWith('enregistrer')).pop();

  // Diagnostic affiché une fois si la fiche n'est pas reconnue
  let diagnosticFait = false;
  function diagnostic() {
    if (diagnosticFait) return; diagnosticFait = true;
    const labels = [...document.querySelectorAll('label')].filter(visible).map(l => l.textContent.trim()).filter(Boolean);
    const boutons = [...document.querySelectorAll('button, a.btn, input[type=submit]')].filter(visible)
      .map(b => (b.textContent || b.value || '').trim()).filter(Boolean);
    consoleOrig.warn('🔍 Diagnostic — la fiche n’est pas reconnue. Copie-moi ces deux listes :\n' +
      '   Libellés visibles : ' + JSON.stringify(labels.slice(0, 50)) + '\n' +
      '   Boutons visibles  : ' + JSON.stringify(boutons.slice(0, 30)));
  }
  // Fiche ouverte = bouton Enregistrer ou champs de la fiche visibles
  const modaleOuverte = () => !!(boutonEnregistrer() || champ('Prénom') || champEmail());

  // Point d'ancrage dans la fiche, pour trouver SA croix (et pas celle du panneau de détail)
  const ancreFiche = () => boutonEnregistrer() || champEmail() || champ('Prénom') || champCommencant('date de naissance');
  const SEL_CROIX = '.blazor-modal__header__closeBtn, [class*="closeBtn"], [class*="close-btn"], .btn-close, button.close, ' +
    '[aria-label="Close"], [aria-label="Fermer"], [title="Fermer"]';
  // Sécurité : jamais un bouton de suppression
  const sansDanger = e => !/delete|supprim|remove/i.test((e.getAttribute('onclick') || '') + ' ' + (e.title || ''));
  // On remonte depuis la fiche jusqu'au premier conteneur qui contient un bouton de fermeture
  function dansFiche(trouver) {
    const ancre = ancreFiche();
    for (let n = ancre?.parentElement; n && n !== document.body; n = n.parentElement) {
      const r = trouver(n); if (r) return r;
    }
    return null;
  }
  const croixFermer = () =>
    dansFiche(n => [...n.querySelectorAll(SEL_CROIX)].find(e => visible(e) && sansDanger(e))) ||
    [...document.querySelectorAll('.blazor-modal__header__closeBtn')].filter(visible).pop();
  const lienFermer = () =>
    dansFiche(n => [...n.querySelectorAll('a, button')].find(a => visible(a) && norm(a.textContent) === 'fermer')) ||
    [...document.querySelectorAll('a, button')].find(a => visible(a) && norm(a.textContent) === 'fermer');

  // Ferme la fiche : croix, puis lien "Fermer", puis touche Échap
  let diagFermeture = false;
  async function fermerModale() {
    if (!modaleOuverte()) return true;
    const essais = [
      () => croixFermer()?.click(),
      () => lienFermer()?.click(),
      () => { const cible = ancreFiche() || document.body;
              for (const t of ['keydown', 'keyup']) cible.dispatchEvent(new KeyboardEvent(t, { key: 'Escape', code: 'Escape', keyCode: 27, bubbles: true })); },
      () => croixFermer()?.click(),
    ];
    for (const essai of essais) {
      essai();
      if (await attendre(() => !modaleOuverte(), 4000)) return true;
    }
    if (!diagFermeture) {
      diagFermeture = true;
      const boutons = [...document.querySelectorAll('button, a, i, span, div')].filter(e => visible(e) &&
        /close|fermer|xmark|times/i.test((e.className || '') + ' ' + (e.title || '') + ' ' + (e.getAttribute?.('aria-label') || '') + ' ' + (e.children.length ? '' : e.textContent)))
        .slice(0, 15).map(e => e.outerHTML.slice(0, 160));
      consoleOrig.warn('🔍 La fiche ne se ferme pas. Éléments de fermeture visibles (copie-les-moi) :\n   ' + boutons.join('\n   '));
    }
    return false;
  }

  /* ---------- Surveillance de la connexion Blazor (WebSocket) ---------- */
  // Blazor Server peut perdre sa connexion (erreur 1006) puis se reconnecter tout seul.
  // On espionne ses messages pour savoir quand c'est coupé / revenu.
  const conn = { connecte: true, coupures: 0 };
  const signalerCoupure = () => { if (conn.connecte) conn.coupures++; conn.connecte = false; };
  const analyser = args => {
    const txt = args.map(a => (a && a.message) || String(a)).join(' ');
    if (/WebSocket closed|Connection disconnected|Cannot send data|not in the 'Connected' State/i.test(txt)) signalerCoupure();
    else if (/WebSocket connected/i.test(txt)) conn.connecte = true;
  };
  const consoleOrig = { error: console.error, info: console.info, log: console.log, warn: console.warn };
  for (const k of Object.keys(consoleOrig)) console[k] = (...a) => { try { analyser(a); } catch (e) {} consoleOrig[k].apply(console, a); };
  const surRejet = e => { try { analyser([e.reason]); } catch (x) {} };
  window.addEventListener('unhandledrejection', surRejet);
  const restaurerConsole = () => { Object.assign(console, consoleOrig); window.removeEventListener('unhandledrejection', surRejet); };

  // État affiché par la fenêtre de reconnexion de Blazor (si elle existe)
  function etatModaleReconnexion() {
    const m = document.getElementById('components-reconnect-modal');
    if (!m) return 'ok';
    const c = m.className;
    if (/rejected|resume-failed|components-reconnect-failed/.test(c)) return 'perdue';
    if (/components-reconnect-(show|retrying|paused)/.test(c)) return 'reconnexion';
    return 'ok';
  }

  // Attend que la connexion soit rétablie. Renvoie false si elle est perdue pour de bon.
  async function attendreConnexion(timeout = 90000) {
    if (conn.connecte && etatModaleReconnexion() === 'ok') return true;
    console.warn('🔌 Connexion au serveur perdue, attente de la reconnexion…');
    const t0 = Date.now();
    while (Date.now() - t0 < timeout) {
      const etat = etatModaleReconnexion();
      if (etat === 'perdue') return false;
      if (conn.connecte && etat === 'ok') { await sleep(2000); console.warn('🔌 Reconnecté, on reprend.'); return true; }
      await sleep(500);
    }
    return false;
  }

  /* ---------- Recherche dans la liste "Associés / Participants" ---------- */
  const conteneurListe = () => document.getElementById('tabMembers1') || document.getElementById('boxMembers') || document;
  const itemsListe = () => [...conteneurListe().querySelectorAll('.memberitem')];
  // Le <span> qui porte le nom (celui avec du texte, pas le rond avec l'icône)
  const spanNom = it => [...it.querySelectorAll('span[onclick*="siderbarDetailsMemberHTML"]')].find(s => s.textContent.trim());
  const nomItem = it => (spanNom(it)?.textContent || '').replace(/\s+/g, ' ').trim();
  const roleItem = it => (it.querySelector('.badge')?.textContent || '').trim();

  // Identifiant du participant, lu dans siderbarDetailsMemberHTML(XX, 10001)
  function idFiche(it) {
    const oc = spanNom(it)?.getAttribute('onclick') || it.querySelector('[onclick*="editCompanyMember("]')?.getAttribute('onclick') || '';
    const m = oc.match(/\(\s*[^,]+,\s*(\d+)/);
    return m ? m[1] : 'nom:' + nomItem(it);
  }

  // Recherche avec le champ "Personnes à chercher" de la liste (#txtSearchMembers)
  const champRecherche = () => document.getElementById('txtSearchMembers');
  const chargementEnCours = () => { const l = document.getElementById('preloadMembers'); return l && l.style.display !== 'none' && visible(l); };
  async function rechercher(terme) {
    const inp = champRecherche();
    if (!inp) return false;
    if (inp.value === terme) return true;
    inp.value = terme;
    if (typeof window.changeTextSearchMembers === 'function') window.changeTextSearchMembers();
    else inp.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true }));
    await sleep(400);
    await attendre(() => !chargementEnCours(), 15000);
    await sleep(300);
    return true;
  }

  // Comparaison des noms : exacte (insensible accents/casse/ordre/tirets), puis tolérante
  const mots = s => norm(s).split(' ').filter(Boolean);
  function lignesDe(cleNom) {
    const tous = itemsListe().filter(it => spanNom(it));
    const exacts = tous.filter(it => cle(nomItem(it)) === cleNom);
    if (exacts.length) return exacts;
    const motsCsv = cleNom.split(' ');
    return tous.filter(it => { const m = new Set(mots(nomItem(it))); return motsCsv.every(x => m.has(x)); });
  }
  function nomsProches(nom) {
    const motsNom = mots(nom);
    return [...new Set(itemsListe().map(nomItem)
      .filter(t => t && motsNom.some(x => x.length > 2 && mots(t).includes(x))))].slice(0, 5);
  }

  // Bouton "Modifier la fiche" du panneau de détail
  const boutonModifierFiche = () =>
    [...document.querySelectorAll('a[onclick*="editCompanyMemberDuringDetailsCompany"]')].find(visible) ||
    [...document.querySelectorAll('a.btn')].find(a => visible(a) && norm(a.textContent) === 'modifier la fiche');

  // Clic sur le nom → panneau de détail → "Modifier la fiche" (repli : crayon de la liste)
  async function ouvrirFiche(it) {
    const ancien = boutonModifierFiche();
    const ocAncien = ancien?.getAttribute('onclick') ?? null;
    if (ancien) ancien.dataset.majVu = '1'; // pour ne pas recliquer sur le panneau de la personne précédente
    if (it.scrollIntoView) it.scrollIntoView({ block: 'center' });
    spanNom(it).click();
    const btn = await attendre(() => {
      const b = boutonModifierFiche();
      return b && (!b.dataset.majVu || b.getAttribute('onclick') !== ocAncien) && b;
    }, 8000);
    if (btn) { btn.click(); return 'panneau'; }
    const crayon = it.querySelector('[onclick*="editCompanyMember("]');
    if (!crayon) return null;
    consoleOrig.warn('   ↪ bouton "Modifier la fiche" non trouvé : ouverture par le crayon de la liste');
    crayon.click();
    return 'crayon';
  }

  /* ---------- 3. Lecture du CSV ---------- */
  const lignesCsv = CSV.split(/\r?\n/).filter(l => l.trim());
  const sep = [';', '\t', ','].find(s => lignesCsv[0].includes(s));
  const decouper = l => l.split(sep).map(c => c.trim().replace(/^"|"$/g, ''));
  // Ligne d'en-tête facultative : si absente, on utilise l'ordre standard des colonnes
  const ENTETES_PAR_DEFAUT = ['Type de personne', 'Prénom', 'Nom', 'Genre', 'Section de vote', 'Section départemental', 'Titulaire / suppléant'];
  const aEntete = norm(lignesCsv[0]).includes('prenom');
  const entetes = (aEntete ? decouper(lignesCsv[0]) : ENTETES_PAR_DEFAUT).map(norm);
  const col = n => entetes.indexOf(norm(n));
  if (!aEntete) console.log('ℹ️ Pas de ligne d’en-tête : ordre des colonnes standard utilisé');

  // Colonnes à corriger = celles présentes dans le CSV (les autres ne sont pas touchées)
  const A_CORRIGER = {
    genre: col('Genre') >= 0,
    section: col('Section de vote') >= 0,
    titulaire: col('Titulaire / suppléant') >= 0,
    naissance: col('Date de naissance') >= 0,
  };
  if (col('Prénom') < 0 || col('Nom') < 0) { console.error('⛔ Le CSV doit contenir les colonnes Prénom et Nom.'); return; }
  console.log('🛠️ Champs corrigés : ' + ([A_CORRIGER.genre && 'Genre', A_CORRIGER.section && 'Section de vote',
    A_CORRIGER.titulaire && 'Titulaire / suppléant', A_CORRIGER.naissance && 'Date de naissance'].filter(Boolean).join(', ') || 'aucun'));
  // Valeur d'une colonne facultative : null = colonne absente ou cellule vide → champ non modifié
  const valeur = (c, nomCol) => { const i = col(nomCol); const v = i >= 0 ? (c[i] || '').trim() : ''; return v || null; };

  let personnes = [];
  for (const l of lignesCsv.slice(aEntete ? 1 : 0)) {
    const c = decouper(l);
    const prenom = c[col('Prénom')] || '', nom = c[col('Nom')] || '';
    if (!(prenom + nom).trim()) continue;
    const genre = (valeur(c, 'Genre') || '').toUpperCase() || null;
    // Contrôle de la ligne : nombre de colonnes et genre H/F
    let erreurCsv = '';
    if (c.length < entetes.length) erreurCsv = `il manque ${entetes.length - c.length} colonne(s)`;
    else if (genre && !['H', 'F'].includes(genre)) erreurCsv = `genre "${genre}" invalide (H ou F attendu)`;
    // Date de naissance : convertie en AAAA-MM-JJ (accepte 05/03/1972, 5/3/72, 1972-03-05, n° de série Excel)
    const naissanceBrute = valeur(c, 'Date de naissance');
    const naissance = naissanceBrute ? dateISO(naissanceBrute) : null;
    if (!erreurCsv && naissanceBrute && !naissance) erreurCsv = `date de naissance "${naissanceBrute}" illisible (JJ/MM/AAAA attendu)`;
    personnes.push({
      num: personnes.length + 1, // n° de la personne dans le CSV (hors en-tête)
      brut: l.trim(),            // ligne d'origine, pour le CSV des échecs
      nom,
      nomCsv: (prenom + ' ' + nom).trim(),
      genre,
      sectionVote: valeur(c, 'Section de vote'),
      titulaire: valeur(c, 'Titulaire / suppléant'),
      naissance,
      email: (valeur(c, 'Email') || valeur(c, 'E-mail') || '').toLowerCase() || null,
      erreurCsv,
    });
    if (erreurCsv) console.warn(`⚠️ Ligne ${personnes.length} du CSV ignorée (${erreurCsv}) : ${l.trim()}`);
  }
  console.log(`📄 ${personnes.length} personne(s) lue(s) dans le CSV`);
  const TOTAL = personnes.length;
  if (CONFIG.DEPART > 1) {
    personnes = personnes.filter(p => p.num >= CONFIG.DEPART);
    console.log(`▶️ Reprise à la ligne ${CONFIG.DEPART}/${TOTAL}`);
  }
  if (CONFIG.LIMITE > 0) personnes = personnes.slice(0, CONFIG.LIMITE);

  /* ---------- 4. Mise à jour d'une fiche (ligne déjà trouvée) ---------- */
  async function traiterLigne(tr, d, res) {
    const ouverture = await ouvrirFiche(tr);
    if (!ouverture) { res.statut = '❌ Impossible d’ouvrir la fiche'; return; }
    res.ouverture = ouverture;

    // Fiche reconnue dès qu'un de ces éléments apparaît (le bouton Enregistrer est revérifié avant d'enregistrer)
    const pret = await attendre(() => boutonEnregistrer() || champ('Prénom') || champCommencant('prenom') ||
      champEmail() || champCommencant('date de naissance'), 15000);
    if (!pret) { diagnostic(); res.statut = '❌ Fiche non ouverte'; await fermerModale(); return; }
    await attendre(() => champ('Nom')?.value, 3000);
    await sleep(CONFIG.DELAI);

    // "Activer les changements" dès l'ouverture de la fiche
    activerModification();
    await sleep(CONFIG.DELAI);
    const selGenre = champ('Genre', 'select');

    // Vérifier qu'on est sur la bonne personne
    const nomFiche = (champ('Prénom')?.value || '') + ' ' + (champ('Nom')?.value || '');
    if (nomFiche.trim() && cle(nomFiche) !== cle(d.nomCsv)) {
      res.statut = `❌ Fiche ouverte = "${nomFiche.trim()}"`;
      await fermerModale(); return;
    }

    // Vérifier l'e-mail de la fiche (si le CSV a une colonne Email remplie)
    if (d.email) {
      const inMail = await attendre(() => champEmail(), 3000);
      if (!inMail) { res.statut = '❌ Champ E-mail introuvable sur la fiche'; await fermerModale(); return; }
      const mailFiche = (inMail.value || '').trim().toLowerCase();
      res.email = mailFiche === d.email ? 'OK' : (mailFiche || '∅');
      if (mailFiche !== d.email) {
        inMail.style.setProperty('border', '2px solid #dc3545', 'important');
        await sleep(CONFIG.DELAI);
        await fermerModale();
        res.statut = mailFiche
          ? `⛔ E-mail différent (fiche : ${mailFiche}) — non modifié`
          : '⛔ E-mail vide sur la fiche — non modifié';
        return;
      }
      marquer(inMail, true);
    }

    // Vérifier / corriger le genre
    let modifie = false;
    const genreActuel = (selGenre?.selectedOptions[0]?.textContent || '').trim().toUpperCase();
    if (!d.genre) {
      res.genre = '–'; // genre non demandé dans le CSV : on n'y touche pas
    } else if (!selGenre) {
      res.statut = '❌ Champ "Genre" introuvable sur la fiche'; await fermerModale(); return;
    } else if (genreActuel === d.genre) {
      res.genre = `OK (${genreActuel})`;
      marquer(selGenre, true);
    } else if (!CONFIG.CORRIGER_GENRE) {
      res.genre = `≠ fiche ${genreActuel} / CSV ${d.genre} (non corrigé)`;
      selGenre.style.setProperty('border', '2px solid #dc3545', 'important');
    } else {
      modifie = true;
      activerModification();
      const sel = await attendre(() => { const s = champ('Genre', 'select'); return s && !s.disabled && s; }, 5000);
      if (!sel) res.genre = '❌ impossible d’activer la modification';
      else {
        const opt = [...sel.options].find(o => o.textContent.trim().toUpperCase() === d.genre);
        if (opt) { saisir(sel, opt.value); marquer(sel, false); res.genre = `${genreActuel} → ${d.genre}`; }
        else res.genre = `❌ genre "${d.genre}" inconnu`;
      }
      await sleep(CONFIG.DELAI);
    }

    // Date de naissance (fiche principale)
    if (d.naissance) {
      let inDate = await attendre(() => champCommencant('date de naissance'), 3000);
      if (!inDate) {
        res.naissance = '❌ champ introuvable';
        res.statut = '❌ Champ "Date de naissance" introuvable'; await fermerModale(); return;
      }
      const avant = (inDate.value || '').trim();
      const voulu = d.naissance; // AAAA-MM-JJ
      if (dateISO(avant) === voulu) {
        res.naissance = `OK (${dateFR(voulu)})`;
        marquer(inDate, true);
      } else {
        modifie = true;
        if (inDate.disabled || inDate.readOnly) {
          activerModification();
          inDate = await attendre(() => { const i = champCommencant('date de naissance'); return i && !i.disabled && !i.readOnly && i; }, 5000);
        }
        if (!inDate) res.naissance = '❌ impossible d’activer la modification';
        else {
          // input type="date" attend AAAA-MM-JJ ; un champ texte attend JJ/MM/AAAA
          saisir(inDate, inDate.type === 'date' ? voulu : dateFR(voulu));
          inDate.dispatchEvent(new Event('blur', { bubbles: true }));
          marquer(inDate, false);
          res.naissance = `${avant || '∅'} → ${dateFR(voulu)}`;
        }
        await sleep(CONFIG.DELAI);
      }
    } else res.naissance = '–';

    // Onglet "Autres (référentiels, ...)" — et PAS "Autres rôles (invité, CAC...)"
    // Seulement si au moins un champ de cet onglet est à vérifier pour cette personne
    if (d.genre || d.sectionVote || d.titulaire) {
      const onglet = [...document.querySelectorAll('a.nav-item')]
        .find(a => visible(a) && norm(a.textContent).startsWith('autres (referentiels'));
      if (!onglet) { res.statut = '❌ Onglet "Autres" introuvable'; await fermerModale(); return; }
      onglet.click();

      // On attend uniquement les champs nécessaires
      const inSection = d.sectionVote ? await attendre(() => champ('SECTION DE VOTE')) : null;
      const inTitulaire = d.titulaire ? await attendre(() => champ('TITULAIRE / SUPPLEANT')) : null;
      if ((d.sectionVote && !inSection) || (d.titulaire && !inTitulaire)) {
        res.statut = '❌ Champs référentiels introuvables'; await fermerModale(); return;
      }

      // Champ GENRE de l'onglet "Autres" (texte libre)
      if (d.genre) {
        const inGenreAutres = await attendre(() => champ('GENRE'), d.sectionVote || d.titulaire ? 1000 : 5000);
        if (inGenreAutres) {
          const avant = inGenreAutres.value || '∅';
          if (!remplir(inGenreAutres, d.genre)) { modifie = true; res.genreAutres = `${avant} → ${d.genre}`; }
          else res.genreAutres = 'OK';
          await sleep(150);
        } else res.genreAutres = 'champ absent';
      } else res.genreAutres = '–';

      if (d.sectionVote) {
        const avantSection = inSection.value || '∅';
        if (!remplir(inSection, d.sectionVote)) { modifie = true; res.section = `${avantSection} → ${d.sectionVote}`; }
        else res.section = 'OK';
        await sleep(150);
      } else res.section = '–';

      if (d.titulaire) {
        const avantTit = inTitulaire.value || '∅';
        if (!remplir(inTitulaire, d.titulaire)) { modifie = true; res.titulaire = `${avantTit} → ${d.titulaire}`; }
        else res.titulaire = 'OK';
      } else res.titulaire = '–';
    }

    await sleep(Math.max(CONFIG.DELAI, CONFIG.PAUSE_VISUELLE)); // le temps de voir les couleurs

    if (!modifie) {
      await fermerModale();
      res.statut = '🟢 Déjà à jour (fermé sans enregistrer)';
    } else if (CONFIG.DRY_RUN) {
      await fermerModale();
      res.statut = '🧪 Test (non enregistré)';
    } else {
      // Bouton Enregistrer : on lui laisse quelques secondes pour apparaître
      const btn = await attendre(() => boutonEnregistrer(), 5000);
      if (!btn) {
        // Fermer sans enregistrer : la boucle d'essais relancera la fiche depuis le début
        await fermerModale();
        res.statut = '❌ Bouton Enregistrer introuvable (fermé sans enregistrer)';
        return;
      }
      btn.click();
      await sleep(CONFIG.DELAI * 3); // laisser l'enregistrement se faire
      const ferme = await fermerModale(); // puis fermer la fiche
      res.statut = ferme ? '✅ Enregistré' : '✅ Enregistré (⚠️ fiche restée ouverte)';
      if (!ferme) consoleOrig.warn('   ↪ fiche restée ouverte : ferme-la à la main, le script attend 20 s…');
      if (!ferme) await attendre(() => !modaleOuverte(), 20000);
    }
    await sleep(CONFIG.DELAI * 2); // laisser la liste se rafraîchir
  }

  /* ---------- 5. Boucle sur le CSV ---------- */
  const resultats = [];
  // Arrêt propre : taper  stopMaj()  dans la console
  window.STOP_MAJ = false;
  window.stopMaj = () => { window.STOP_MAJ = true; console.warn('⏹️ Arrêt demandé : le script s’arrête après la fiche en cours.'); };
  console.log('ℹ️ Pour arrêter proprement : tapez  stopMaj()  dans la console');
  console.log(`🔎 Liste des participants : ${itemsListe().length} affiché(s)` +
    (champRecherche() ? ' · recherche "Personnes à chercher" utilisée' : ' · ⚠️ champ de recherche introuvable, seuls les participants affichés seront trouvés'));

  let arretConnexion = null;

  for (const d of personnes) {
    if (window.STOP_MAJ) break;
    const cleNom = cle(d.nomCsv);
    const progression = `${d.num}/${TOTAL}`;

    // Ligne du CSV mal formée : on ne touche à rien, elle part dans le CSV des échecs
    if (d.erreurCsv) {
      resultats.push({ ligne: progression, nom: d.nomCsv, statut: `❌ Ligne CSV invalide : ${d.erreurCsv}`, _d: d });
      consoleOrig.log(`[${progression}] ❌ Ligne CSV invalide (${d.erreurCsv}) — ${d.nomCsv}`);
      continue;
    }

    // Ne rien faire tant que la connexion au serveur n'est pas rétablie
    if (!(await attendreConnexion())) { arretConnexion = d.num; break; }

    // Recherche de la personne dans la liste (par son nom de famille)
    await rechercher(d.nom.trim());
    await attendre(() => lignesDe(cleNom).length, 3000);

    const nb = lignesDe(cleNom).length;
    if (!nb) {
      const proches = nomsProches(d.nom);
      resultats.push({ ligne: progression, nom: d.nomCsv,
        statut: '⚠️ Introuvable dans la liste' + (proches.length ? ` (proches : ${proches.join(', ')})` : ''), _d: d });
      consoleOrig.log(`[${progression}] ⚠️ Introuvable — ${d.nomCsv}` +
        (proches.length ? `\n      Noms approchants dans la liste : ${proches.join(' | ')}` : ''));
      continue;
    }

    // Plusieurs lignes peuvent pointer vers le MÊME participant : on ne l'ouvre qu'une fois
    const cles = [...new Set(lignesDe(cleNom).map(idFiche))];
    const aTraiter = CONFIG.TOUTES_LES_LIGNES ? cles.length : 1;
    if (nb > cles.length) consoleOrig.log(`[${progression}] ℹ️ ${nb} lignes dans la liste mais ${cles.length} fiche(s) distincte(s) — ${d.nomCsv}`);
    for (let i = 0; i < aTraiter && !window.STOP_MAJ; i++) {
      // Ligne du CSV / total, + n° de la fiche seulement si la personne a plusieurs fiches distinctes
      const ligne = cles.length > 1 ? `${progression} (fiche ${i + 1}/${cles.length})` : progression;
      let res;
      // Jusqu'à 3 essais si la connexion saute pendant la fiche.
      // Sans risque : une fiche déjà enregistrée ressort "Déjà à jour" au nouvel essai.
      for (let essai = 1; essai <= 3; essai++) {
        res = { ligne, nom: d.nomCsv, role: '', ouverture: '', email: '', genre: '', genreAutres: '', section: '', titulaire: '', naissance: '', statut: '' };
        const coupuresAvant = conn.coupures;
        try {
          if (!(await attendreConnexion())) { res.statut = '❌ Connexion perdue'; break; }
          // On relance la recherche : la liste peut avoir été rafraîchie après l'enregistrement
          await rechercher(d.nom.trim());
          const tr = await attendre(() => lignesDe(cleNom).find(t => idFiche(t) === cles[i]), 4000);
          if (!tr) { res.statut = '❌ Participant disparu de la liste après rafraîchissement'; }
          else {
            res.role = roleItem(tr);
            await traiterLigne(tr, d, res);
          }
        } catch (e) {
          res.statut = '❌ Erreur : ' + e.message;
          consoleOrig.error(d.nomCsv, e);
          await fermerModale();
        }
        const coupure = conn.coupures !== coupuresAvant;
        if (!coupure && !res.statut.startsWith('❌')) break; // tout s'est bien passé
        if (essai < 3) {
          consoleOrig.warn(`[${ligne}] ${coupure ? '🔌 Coupure pendant la fiche' : res.statut} — nouvel essai (${essai + 1}/3)…`);
          if (!(await attendreConnexion())) break;
          await fermerModale();
          await sleep(1500);
        } else if (coupure) {
          res.statut = '⚠️ À vérifier (coupure de connexion) — ' + res.statut;
        }
      }
      res._d = d;
      // Homonyme : si une AUTRE fiche du même nom a le bon e-mail, celle-ci est juste ignorée
      if (res.statut.startsWith('⛔') && cles.length > 1) res.statut = '⏭️ Homonyme ignoré (e-mail différent)';
      resultats.push(res);
      consoleOrig.log(`[${res.ligne}] ${res.statut} — ${d.nomCsv}`);
      if (res.statut === '❌ Connexion perdue') { arretConnexion = d.num; break; }
    }
    // Homonymes : si AUCUNE fiche n'avait le bon e-mail, c'est un échec
    const resPersonne = resultats.filter(r => r._d === d);
    if (resPersonne.length > 1 && resPersonne.every(r => r.statut.startsWith('⏭️'))) {
      for (const r of resPersonne) r.statut = '⛔ Aucune fiche avec cet e-mail — non modifié';
      consoleOrig.warn(`[${progression}] ⛔ Aucune des ${resPersonne.length} fiches n'a l'e-mail ${d.email} — ${d.nomCsv}`);
    }
    if (arretConnexion) break;
  }

  restaurerConsole();
  if (arretConnexion) {
    console.error(`⛔ Connexion au serveur perdue définitivement à la ligne ${arretConnexion}/${TOTAL}.\n` +
      `   Recharge la page (F5), puis relance le script avec  DEPART: ${arretConnexion}`);
  }
  await rechercher(''); // on vide la recherche

  /* ---------- 6. CSV des lignes en échec (erreur, introuvable, à vérifier, non traitées) ---------- */
  const echecs = resultats.filter(r => /^(❌|⚠️|⛔)/.test(r.statut));
  const traites = new Set(resultats.map(r => r._d.num));
  const nonTraites = personnes.filter(p => !traites.has(p.num)); // si arrêt en cours de route
  const lignesExport = [
    ...echecs.map(r => ({ d: r._d, statut: r.statut, table: r.ouverture || '', role: r.role || '' })),
    ...nonTraites.map(d => ({ d, statut: '⏸️ Non traité (script arrêté)', table: '', role: '' })),
  ].sort((a, b) => a.d.num - b.d.num);

  if (lignesExport.length) {
    const propre = s => String(s).replace(/[;\r\n]+/g, ' ').trim();
    const entete = (aEntete ? lignesCsv[0].trim().split(sep).join(';') : ENTETES_PAR_DEFAUT.join(';'))
      + ';N° ligne;Statut;Ouverture;Rôle';
    const corps = lignesExport.map(x =>
      [x.d.brut.split(sep).join(';'), x.d.num, propre(x.statut), propre(x.table), propre(x.role)].join(';'));
    const contenu = '﻿' + [entete, ...corps].join('\r\n'); // BOM pour les accents dans Excel
    const horodatage = new Date().toISOString().slice(0, 16).replace(/[-:T]/g, '');
    const lien = document.createElement('a');
    lien.href = URL.createObjectURL(new Blob([contenu], { type: 'text/csv;charset=utf-8' }));
    lien.download = `maj_membres_v2_echecs_${horodatage}.csv`;
    document.body.appendChild(lien); lien.click(); lien.remove();
    console.warn(`📥 ${lignesExport.length} ligne(s) en échec exportée(s) dans ${lien.download}`);
    console.log([entete, ...corps].join('\n')); // copie de secours dans la console
  } else {
    console.log('🎉 Aucune ligne en échec.');
  }

  console.table(resultats.map(({ _d, ...r }) => r));
})();```
