# Minihell (minishell)

> Un shell Unix minimaliste en C, inspiré de bash : prompt interactif avec historique, pipes, redirections, heredoc, expansion des variables et builtins.

*« Collaboration invraisemblable entre deux ennemis jurés : l'aigri originel et Le Z s'allient pour éviter d'avoir à faire #Minihell avec des gens barbants. »*

## À propos

`minishell` est un projet de groupe du cursus de l'école 42. Il consiste à réécrire un petit interpréteur de commandes qui se comporte comme bash sur un sous-ensemble de fonctionnalités : lire une ligne, la découper en tokens, la parser en commandes, développer les variables puis exécuter le tout avec `fork`, `execve`, `pipe` et `dup2`.

**Projet réalisé à deux :**
- **[MarkusLeGrand](https://github.com/MarkusLeGrand)** : lexer, parser, expansion des variables et des quotes, gestion de `;`, mise à la norme ;
- **[Zetsudes](https://github.com/Zetsudes)** (login 42 : `zamohame`) : exécution, pipes, redirections et heredoc, builtins, environnement, signaux, codes de sortie.

Ce dépôt est un fork du dépôt d'origine [Zetsudes/Minigreg](https://github.com/Zetsudes/Minigreg).

## Fonctionnalités

- **Prompt interactif** `🔥 minihell$ ` avec historique, grâce à GNU readline.
- **Mode non interactif** : si l'entrée standard n'est pas un terminal (par exemple `echo "ls" | ./minishell`), les lignes sont lues sur `stdin`.
- **Quotes** :
  - `'...'` : aucune interprétation ;
  - `"..."` : `$` est développé ;
  - backslash (`\`) pris en charge ;
  - une quote non fermée provoque une erreur de syntaxe.
- **Expansion** de `$VAR` et de `$?` (code de sortie de la dernière commande). Un argument non quoté qui devient vide après expansion est supprimé, comme dans bash.
- **Redirections** :
  - `<` : entrée depuis un fichier ;
  - `>` : sortie dans un fichier (écrasé) ;
  - `>>` : sortie ajoutée à la fin du fichier ;
  - `<<` : heredoc.
- **Pipes** `|`, en nombre illimité.
- **Séparateur `;`** pour enchaîner plusieurs commandes ou pipelines.
- **Builtins** :
  - `echo` (avec `-n`, y compris `-nnn`) ;
  - `cd` (sans argument : `$HOME` ; `cd -` ; mise à jour de `PWD` et `OLDPWD`) ;
  - `pwd` ;
  - `export` (sans argument : affiche les variables triées au format `declare -x`) ;
  - `unset` ;
  - `env` ;
  - `exit` (vérifie que l'argument est numérique, code modulo 256, refuse plusieurs arguments).
- **Signaux** :
  - `Ctrl-C` affiche un nouveau prompt propre ;
  - `Ctrl-\` est ignoré ;
  - `Ctrl-D` quitte le shell.
- **Codes de sortie fidèles à bash** :
  - `127` : commande introuvable ;
  - `126` : permission refusée, ou le chemin est un dossier ;
  - `2` : erreur de syntaxe.
- **Messages d'erreur de syntaxe** au format de bash : `syntax error near unexpected token`.
- **Environnement minimal** (`PWD`, `PATH`, `HOME`, `SHLVL`) si le shell est lancé avec un environnement vide (`env -i ./minishell`).

## Compilation & utilisation

Prérequis : `gcc`, `make` et la bibliothèque **readline** (`sudo apt install libreadline-dev` sur Debian/Ubuntu).

```bash
make        # compile la libft du groupe (greg_lib/) puis le binaire ./minishell
make clean  # supprime les fichiers objets
make fclean # supprime aussi le binaire et la libft
make re     # recompile
```

Exemple de session :

```
$ ./minishell
🔥 minihell$ export NAME=42
🔥 minihell$ echo "Hello $NAME" | cat -e
Hello 42$
🔥 minihell$ echo '$NAME' "$NAME"
$NAME 42
🔥 minihell$ ls nope ; echo $?
ls: cannot access 'nope': No such file or directory
2
🔥 minihell$ cat << EOF > out.txt
heredoc> ligne 1
heredoc> EOF
🔥 minihell$ cat < out.txt | wc -l >> count.txt
🔥 minihell$ exit
exit
```

## Approche technique

Une ligne passe par quatre étapes : **lexer → parser (avec expansion) → exécution → mise à jour de `$?`**.

1. **Lexer** (`lexer/`). Un automate à trois états (`GENERAL`, `IN_SQ`, `IN_DQ`) découpe la ligne en tokens. En dehors des quotes, les espaces séparent les mots et les opérateurs `<`, `>`, `<<`, `>>`, `|` et `;` deviennent des tokens à part. Les quotes sont conservées dans les tokens pour l'étape d'expansion. Les buffers grandissent dynamiquement, sans `realloc`, qui est interdit par le sujet.
2. **Parser** (`parsing/`).
   - Les tokens sont regroupés en une liste chaînée de `t_cmd`. Chaque commande contient ses arguments, `infile`, `outfile`, ainsi que les drapeaux `append`, `heredoc` et `cmd_separator` (`;`).
   - Les erreurs de syntaxe sont détectées à cette étape : pipe en début ou en fin de ligne, redirection sans fichier, `;` doublé.
   - Les fichiers de redirection sont ouverts dès le parsing, comme le fait bash, qui crée les fichiers de sortie même si la commande échoue.
3. **Expansion** (`expand_var/`). Chaque token est réécrit caractère par caractère. Un contexte suit l'état des quotes simples et doubles : `$VAR` et `$?` sont remplacés, les quotes sont retirées et les échappements `\` sont traités selon le contexte.
4. **Exécution** (`exec/`, `pipe/`).
   - La liste est d'abord découpée en groupes séparés par `;`.
   - Pour une commande seule, un builtin s'exécute dans le processus parent : les descripteurs `stdin` et `stdout` sont sauvegardés avec `dup`, redirigés avec `dup2`, puis restaurés. Sinon, le shell fait `fork` puis `execve`.
   - Pour un pipeline de *n* commandes, le shell crée *n−1* pipes, puis fait un `fork` par commande et branche les descripteurs avec `dup2`. Les builtins tournent alors dans le processus enfant.
   - Le parent ferme tous les pipes puis attend tous les enfants. Le code de sortie retenu est celui de la dernière commande.
   - Le binaire est cherché dans `PATH`. Un chemin contenant `/` est utilisé tel quel.
   - Le heredoc lit l'entrée avec `get_next_line` et l'écrit dans un pipe, qui devient le `stdin` de la commande.
5. **Environnement** (`env/`). Les variables sont stockées dans une liste chaînée clé/valeur, convertie en `char **` au moment d'appeler `execve`. `$?` y est stocké sous la clé `?`, que `env` n'affiche pas.

## Structure du projet

```
Minigreg/
├── Makefile
├── include/        # en-têtes par module (lexer, parsing, exec, pipe, builtins, env, signals...)
├── lexer/          # découpage de la ligne en tokens
├── parsing/        # construction des t_cmd, redirections, erreurs de syntaxe
├── expand_var/     # expansion de $VAR / $? et gestion des quotes
├── exec/           # main, boucle readline, exécution, PATH, redirections, heredoc
├── pipe/           # pipelines multi-commandes et nettoyage
├── builtins/       # echo, cd, pwd, export, unset, env, exit
├── env/            # environnement en liste chaînée
├── signals/        # Ctrl-C, Ctrl-\
└── greg_lib/       # libft du groupe + ft_printf + get_next_line
```

## Ce que j'ai appris

- Écrire un lexer à états et un parser capables de reproduire les règles de quoting de bash, avec tous leurs cas limites.
- Comprendre en profondeur `fork`, `execve`, `waitpid`, `pipe` et `dup2`, et l'importance de fermer chaque descripteur au bon endroit pour éviter les blocages.
- Distinguer ce qui doit s'exécuter dans le shell lui-même (`cd`, `export`, `exit`) de ce qui peut tourner dans un processus enfant.
- Gérer les signaux dans un programme interactif avec readline.
- Reproduire les codes de sortie et les messages d'erreur de bash, en comparant systématiquement avec testeurs et cas limites.
- Travailler à deux sur une base de code C : découpage en modules, interfaces communes (`t_cmd`, `t_env`), fusions Git et respect de la Norme 42.
