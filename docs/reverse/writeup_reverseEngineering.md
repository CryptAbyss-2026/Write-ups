# Write-up : Reverse Engineering (Symboles)

## Description
Le challenge fournit un fichier objet (`chall.o`) contenant une fonction de flag qui n'est jamais appelée.

## Analyse
L'outil `nm` permet d'inspecter la table des symboles du fichier objet. On y découvre une fonction nommée `print_flag`. Le fichier n'étant pas un exécutable complet (absence de `main`), il ne peut pas être lancé tel quel.

## Résolution
Il faut créer un programme "wrapper" en C qui déclare la fonction comme externe et l'appelle dans un `main`, puis compiler l'ensemble en liant le fichier objet original.

## Commandes

### 1. Créer solve.c
```c
extern void print_flag();
int main() {
    print_flag();
    return 0;
}
```

### 2. Compiler et exécuter
```bash
gcc solve.c chall.o -o solve
./solve
```
