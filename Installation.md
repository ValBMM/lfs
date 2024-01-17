# Installation d’un Linux from Scratch avec Luks et LVM

# Sommaire

[Introduction](Installation%20d%E2%80%99un%20Linux%20from%20Scratch%20avec%20Luks%20et%20%20df5021cb38704705acefc27ecd2a9146.md) 

[Prérequis](Installation%20d%E2%80%99un%20Linux%20from%20Scratch%20avec%20Luks%20et%20%20df5021cb38704705acefc27ecd2a9146.md) 

[Préparation du système pilote](Installation%20d%E2%80%99un%20Linux%20from%20Scratch%20avec%20Luks%20et%20%20df5021cb38704705acefc27ecd2a9146.md) 

Partitionnement du disque cible

Préparation de l’arborescence 

Compilation croisée des paquets

Préparation du système cible avec chroot

Installation des paquets

Configuration du système

Configuration du boot

# Introduction

Le but de ce projet est de voir comment installer un système Linux from scratch (=depuis 0) avec systemd grâce au livre LFS de **Gerard Beekmans (**[https://www.fr.linuxfromscratch.org/](https://www.fr.linuxfromscratch.org/)). Cette méthodologie/documentation nous indique comment installer un système Linux de A à Z, pour notre part nous allons prendre quelques initiatives qui demandent plus de rigueur et d’attention en partitionnant notre disque dur avec LVM et en le chiffrant avec Luks. Ces 2  parties ne sont pas présentes dans le livre et demandent donc une connaissance du sujet.

# Prérequis

L’installation d’un Linux from Scratch (que nous abrévierons LFS) n’est pas ouvert à toutes personnes, il demande des connaissances des systèmes Linux car il est facile en procédant à la création de son LFS de casser son système actuel.

L’installation d’un LFS nécessite en effet un système Linux que nous allons utiliser pour piloter l’installation, Il nous servira à partitionner notre disque, récupérer et compiler les dépendances et monter notre arborescence, il sera **système pilote** et la **cible sera là ou nous installerons le LFS**

Sur notre système pilote nous avons besoin de nous assurer que certaines dépendances sont présentes car elles nous seront indispensables à l’installation. Le script ci-dessous vérifie la présence et la version des paquets nécessaire :

```bash
cat > version-check.sh << "EOF"
#!/bin/bash
# Un script qui liste les numéro de version des outils de développement critiques

# Si vous avez des outils installés dans d'autres répertoires, ajustez PATH ici ET
# dans ~lfs/.bashrc (section 4.4) également.

LC_ALL=C 
PATH=/usr/bin:/bin

bail() { echo "FATAL : $1"; exit 1; }
grep --version > /dev/null 2> /dev/null || bail "grep ne fonctionne pas"
sed '' /dev/null || bail "sed ne fonctionne pas"
sort   /dev/null || bail "sort ne fonctionne pas"

ver_check()
{
   if ! type -p $2 &>/dev/null
   then 
     echo "ERREUR : $2 ($1) introuvable"; return 1; 
   fi
   v=$($2 --version 2>&1 | grep -E -o '[0-9]+\.[0-9\.]+[a-z]*' | head -n1)
   if printf '%s\n' $3 $v | sort --version-sort --check &>/dev/null
   then 
     printf "OK :     %-9s %-6s >= $3\n" "$1" "$v"; return 0;
   else 
     printf "ERREUR : %-9s est TROP VIEUX (version $3 ou supérieure requise)\n" "$1"; 
     return 1; 
   fi
}

ver_kernel()
{
   kver=$(uname -r | grep -E -o '^[0-9\.]+')
   if printf '%s\n' $1 $kver | sort --version-sort --check &>/dev/null
   then 
     printf "OK :     noyau Linux $kver >= $1\n"; return 0;
   else 
     printf "ERREUR : noyau Linux ($kver) est TROP VIEUX (version $1 ou supérieure requise)\n" "$kver"; 
     return 1; 
   fi
}

# Coreutils en premier car --sort a besoin de Coreutils >= 7.0
ver_check Coreutils      sort     7.0 || bail "--version-sort n'est pas pris en charge"
ver_check Bash           bash     3.2
ver_check Binutils       ld       2.13.1
ver_check Bison          bison    2.7
ver_check Diffutils      diff     2.8.1
ver_check Findutils      find     4.2.31
ver_check Gawk           gawk     4.0.1
ver_check GCC            gcc      5.1
ver_check "GCC (C++)"    g++      5.1
ver_check Grep           grep     2.5.1a
ver_check Gzip           gzip     1.3.12
ver_check M4             m4       1.4.10
ver_check Make           make     4.0
ver_check Patch          patch    2.5.4
ver_check Perl           perl     5.8.8
ver_check Python         python3  3.4
ver_check Sed            sed      4.1.5
ver_check Tar            tar      1.22
ver_check Texinfo        texi2any 5.0
ver_check Xz             xz       5.0.0
ver_kernel 4.14

if mount | grep -q 'devpts on /dev/pts' && [ -e /dev/ptmx ]
then echo "OK :     le noyau Linux prend en charge les PTY UNIX 98";
else echo "ERREUR : le noyau Linux ne prend PAS en charge les PTY UNIX 98"; fi

alias_check() {
   if $1 --version 2>&1 | grep -qi $2
   then printf "OK :     %-4s est $2\n" "$1";
   else printf "ERREUR : %-4s n'est PAS $2\n" "$1"; fi
}
echo "Alias :"
alias_check awk GNU
alias_check yacc Bison
alias_check sh Bash

echo "Vérification du compilateur :"
if printf "int main(){}" | g++ -x c++ -
then echo "OK :     g++ fonctionne";
else echo "ERREUR : g++ ne fonctionne PAS"; fi
rm -f a.out
EOF

bash version-check.sh
```

*“EOF = End of Line”*

# Préparation du système pilote

Avant de commencer nous allons préparer notre système pilote afin de nous facilité dans la suite.

Nous allons créer des variables pour simplifier les commandes :

```bash
export “LFS=/mnt/lfs” \
export “SWAP=/dev/mapper/secure-swap” \
export “SYSTEM=/dev/mapper/secure-system”

echo $LFS
> /mnt/lfs
echo $SWAP
> /dev/mapper/secure-swap
echo $SYSTEM
> /dev/mapper/secure-system
```

 Ces variables nous servirons pour créer notre arborescence.

Nous allons aussi créer un utilisateur pour préparer notre système cible :

```bash
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```