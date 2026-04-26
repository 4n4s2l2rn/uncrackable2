# 🔓 OWASP MAS Crackme — Android UnCrackable Level 2
## Writeup — Analyse Statique avec jadx (sans Frida)

**Auteur :** Writeup personnel  
**Outil principal :** jadx-gui  
**Difficulté :** Medium  
**Objectif :** Trouver le mot de passe secret de l'application

---

## 📋 Description du challenge

L'application affiche un champ texte et demande un secret. Elle implémente plusieurs protections :
- Détection de root
- Détection de debugger
- Logique de vérification dans une bibliothèque native (`libfoo.so`)

---

## 🛠️ Outils utilisés

| Outil | Usage |
|-------|-------|
| jadx-gui 1.5.5 | Décompilation du code Java |
| strings (Sysinternals) | Extraction des chaînes du binaire natif |
| 7-Zip | Extraction de l'APK |

---

## 🔍 Étape 1 — Analyse statique avec jadx-gui

### 1.1 Ouvrir l'APK

Lancer `jadx-gui` puis :
```
File → Open files → UnCrackable-Level2.apk
```

### 1.2 Lire MainActivity.java

Dans le panneau gauche :
```
Source code → sg.vantagepoint.uncrackable2 → MainActivity
```

**Code important trouvé :**

```java
static {
    System.loadLibrary("foo");  // charge libfoo.so
}

protected void onCreate(Bundle bundle) {
    init();

    // Protection 1 — Root detection
    if (b.a() || b.b() || b.c()) {
        a("Root detected!");
    }

    // Protection 2 — Debug detection
    if (a.a(getApplicationContext())) {
        a("App is debuggable!");
    }

    // Protection 3 — Debugger attaché (boucle infinie)
    new AsyncTask<Void, String, String>() {
        public String doInBackground(Void... voidArr) {
            while (!Debug.isDebuggerConnected()) {
                SystemClock.sleep(100L);
            }
            return null;
        }
        public void onPostExecute(String str) {
            MainActivity.this.a("Debugger detected!");
        }
    }.execute(null, null, null);
}

public void verify(View view) {
    String string = ((EditText) findViewById(R.id.edit_text)).getText().toString();
    if (this.m.a(string)) {  // appelle CodeCheck.a()
        // Succès
    }
}
```

**Observation :** La méthode `a()` dans MainActivity appelle `System.exit(0)` si root/debug détecté. La vérification du mot de passe passe par `CodeCheck`.

---

### 1.3 Lire CodeCheck.java

Dans le panneau gauche :
```
Source code → sg.vantagepoint.uncrackable2 → CodeCheck
```

**Code trouvé :**

```java
public class CodeCheck {
    private native boolean bar(byte[] bArr);  // méthode NATIVE

    public boolean a(String str) {
        return bar(str.getBytes());  // délègue au code natif
    }
}
```

**Observation clé :** La vérification est dans une méthode **native** `bar()` → le secret est dans `libfoo.so`, jadx ne peut pas le décompiler directement.

---

## 🔍 Étape 2 — Extraire libfoo.so

Un fichier APK est simplement une archive ZIP.

### 2.1 Renommer l'APK en ZIP

```
UnCrackable-Level2.apk → UnCrackable-Level2.zip
```

### 2.2 Extraire avec 7-Zip

Clic droit → 7-Zip → Extraire ici

### 2.3 Naviguer vers la lib native

```
lib/
├── x86/
│   └── libfoo.so
└── x86_64/
    └── libfoo.so     ← prendre celle-ci selon l'architecture
```

Copier `libfoo.so` dans le dossier de travail.

---

## 🔍 Étape 3 — Analyser libfoo.so avec strings

### Sur Windows (Sysinternals strings.exe)

```cmd
strings.exe libfoo.so
```

### Sur Kali Linux

```bash
strings libfoo.so
```

### Résultat obtenu

```
LF
tdx
Android
waitpid
LIBC
libc.so
libfoo.so
__cxa_atexit
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar
Java_sg_vantagepoint_uncrackable2_MainActivity_init
ptrace
strncmp
pthread_create
$Than        ← 🚨 début du secret
ks f         ← 🚨
or a         ← 🚨
ll tf        ← 🚨
 fisf        ← 🚨
```
<img width="1087" height="694" alt="strings" src="https://github.com/user-attachments/assets/6ce20bf5-8325-4fb0-a134-a8dd6967e5ef" />

**Observation :** La chaîne secrète est fragmentée dans le binaire pour rendre l'analyse plus difficile. En assemblant les fragments :

```
$Than + ks f + or a + ll t + h + e + f + ish
→ "Thanks for all the fish"
```

---

## 🧩 Étape 4 — Confirmer le secret

### Fonctionnement de la vérification

D'après l'analyse des imports dans libfoo.so :
```
strncmp     ← utilisé pour comparer l'input avec le secret
ptrace      ← anti-debug
pthread_create ← thread anti-debug
```

La fonction native `bar()` :
1. Reçoit les bytes de l'input utilisateur
2. Compare avec la chaîne secrète via `strncmp`
3. Retourne `true` si égal, `false` sinon

---

## 🏁 Solution

Le mot de passe secret est :

```
Thanks for all the fish
```

> 💡 Référence au roman de science-fiction "Le Guide du voyageur galactique" de Douglas Adams.

---

## 📊 Résumé des protections et contournements

| Protection | Méthode | Contournement (statique) |
|------------|---------|--------------------------|
| Root detection | `b.a()`, `b.b()`, `b.c()` | Non nécessaire (analyse statique) |
| Debug detection | `a.a(context)` | Non nécessaire (analyse statique) |
| Debugger loop | `Debug.isDebuggerConnected()` | Non nécessaire (analyse statique) |
| Secret natif | `libfoo.so` → `strncmp` | `strings libfoo.so` |

---

## 📁 Arborescence des fichiers analysés

```
UnCrackable-Level2.apk
├── classes.dex
│   └── sg.vantagepoint.uncrackable2
│       ├── MainActivity.java   ✅ analysé
│       └── CodeCheck.java      ✅ analysé
└── lib/
    └── x86_64/
        └── libfoo.so           ✅ strings extraites
```

---

## ✅ Conclusion

L'analyse statique seule suffit pour résoudre ce challenge. Sans exécuter l'application ni utiliser Frida, il est possible de :

1. Comprendre la logique de vérification via jadx
2. Identifier que le secret est dans le code natif
3. Extraire le secret avec la commande `strings` sur `libfoo.so`

**Mot de passe trouvé : `Thanks for all the fish`**
on peut utiliser la methode avec guidra plus simple(voir lab5.png)
