---
layout: post
title: 'Caliper me : Ou pourquoi les microbenchmarks aident !'
date: 2012-08-29 00:47:50.000000000
type: post
published: true
status: publish
tags:
- code
- design
- performance
- big o
- cailper
- complexity
- java
- maven
- measure
- memory
- metrics
- microbenchmark
- performance
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
author: Brice Dutheil
disqus_identifier: '362 http://blog.arkey.fr/?p=362'
---

**ATTENTION : Cet article n'est plus d'actualité, pour les microbenchmark il faut maintenant utiliser
[JMH](http://openjdk.java.net/projects/code-tools/jmh/) (développé par [Aleksey Shipilev](http://shipilev.net/),
en ce moment employé par Oracle, 2016-06-14), Caliper et les autres outils de microbenchmark ont de graves problèmes
qui ne permettent pas de faire des microbenchmarks correctes.**

------------------------------------

# Contexte

Dans les news récemment

* Guava 13.0 réduit la consommation en mémoire de plusieurs structures [[source](http://code.google.com/p/guava-libraries/wiki/Release13)]
* J'ai passé des tests d'algorithmie [[source](#)]

Premier point, on sait que Google est attentif sur les performances de ses librairies. Sur le deuxième point, ben j'ai finalement loupé un des exercices "timeboxés", je n'ai pas eu le temps de le finir a temps, du coup le code soumis était 100 % foireux. Mais bon ce n'est pas le sujet du billet, ce qui est important c'est que j'ai fini après cet algo, puis je me suis rendu compte que celui-ci n'était pas optimal, il ressemblait à un algo en ~~O(n²)~~ O(n log n) ([*Merci Sam pour cette correction*](#comment-834)). Du coup j'ai repris le papier et le crayon, et je me suis rendu compte que je pouvais utiliser une technique similaire à celle que j'ai utilisé sur le premier algo du test (comment n'ai-je pas pu y penser à ce moment d'ailleurs ?).

De mémoire l'algo porte grosso modo sur le comptage de paires d'entier distante d'un nombre K dans un tableau A.

Ma première version :

```java
class PairCounts_V1 {
    public int countPairsWithKDistance(int K, int[] A) {
        if(A == null || A.length < 2) return 0;

        Arrays.sort(A);

        int paircounter = 0;

        int j = A.length - 1;
        for (int i = 0; i <= j; i++) {
            int val_i = A[i];

            for(; i < j && K - A[j] != val_i; j--);
            int val_j = A[j];

            if ((long) (val_i + val_j) == K) {
                paircounter = incrementPairCounter(paircounter, i, j);

                // count duplicates
                while(val_j == A[--j]) paircounter =
                              incrementPairCounter(paircounter, i, j);
            }
            j = A.length-1;
        }

        return kpaircounter;
    }

    private int incrementPairCounter(int paircounter, int i, int j) {
        paircounter++;
        if(i != j ) paircounter++;
        return paircounter;
    }
}
```

Alors pourquoi je pense que cet algo n'est pas optimal : simplement du fait des boucles inbriquée, on dirait du ~~O(n²)~~ O(n log n) (*[voir ici pourquoi](#comment-834)*). Mais quand on parle de la performance ou de la complexité d'un algorithme il ne faut prendre uniquement compte de l'invariant mais aussi **du jeu de données : quelle taille ? quelle distribution ? quel type de données ?**

En effet un algorithme pour être efficace doit être adapté aux jeux de données qu'il aura à traiter et à l'usage qu'on en a ; peut-être à travers des paramètres ou des structures différentes, typiquement un des constructeur d'une HashMap prend des paramètres comme [HashMap](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/HashMap.html) si la recherche est un cas d'utilisation de la structure de donnée.

Bref du coup voilà à quoi ressemble la nouvelle version de cet algo :

```java
class PairCounts_V2 {

public int countPairsWithKDistance(int K, int[] A) {
        if(A == null || A.length < 2) return 0;

        HashMap<Long, Integer> complements = new HashMap<Long, Integer>();
        for (int number : A) {
            long complement = K - number;

            Integer complementCount = complements.get(complement);
            complementCount = complementCount == null ?
                                  Integer.valueOf(1)
                                  : ++complementCount;
            complements.put(complement, complementCount);
        }

        int paircounter = 0;
        for (int number : A) {
            Long key = Long.valueOf(number);
            Integer complementCount = complements.get(key);
            paircounter = complementCount == null ?
                              paircounter
                              : paircounter + complementCount;
        }

        return paircounter;
    }

}
```

Donc ici l'idée c'est de préparer dans un premier temps un dictionnaire  inversé basé sur un des entier et la distance demandée, en incrémentant pour chaque occurence. Ici une seule boucle for, car on parcours en entier le tableau. Dans un second temps on cherche les entiers qui correspondent effectivement à l'entrée de ce dictionnaire, et si oui on incrémente le compteur de paires. Là aussi une seule boucle sur le tableau donc O(n). Sachant qu'une HashMap a souvant une complexité de O(1) pour l'insertion et la recherche, a vue de nez l'algo est plutot pas mal.

Bon mais dans la réalité ca donne quoi, en effet comme Kirk Pepperdine et bien d'autres disaient : ***Measure ! Don't guess !***

# Caliper me voilà !

[ici](http://code.google.com/p/guava-libraries/source/browse/#git%2Fguava-tests%2Fbenchmark%2Fcom%2Fgoogle%2Fcommon%2Fcollect) par exemple.

Avec un projet "mavenisé" on récupère la dernière version de **caliper**, la version **0.5-rc1** aujourd'hui.

```xml
<dependency>
    <groupid>com.google.caliper</groupid>
    <artifactid>caliper</artifactid>
    <version>0.5-rc1</version>
</dependency>
```

Pour écrire un benchmark caliper il suffit d'étendre la classe `[@Param](http://caliper.googlecode.com/svn/static/api/reference/com/google/caliper/Param.html)`.

Enfin comme Caliper lance une nouvelle VM, fait quelques travaux pour chauffer la VM (warmup), etc, il faut pour l'instant lancer ces tests avec une commande manuelle :

```sh
java -classpath $THE_CLASSPATH com.google.caliper.Runner PairCountsBenchmark
```

La ligne de commande pourra varier suivant les besoins ; on peut notamment se rendre sur leur site pour y voir les [code](http://code.google.com/p/caliper/source/browse/caliper/src/main/java/com/google/caliper/Runner.java?name=v0.5-rc1)) à passer au `Runner` Caliper. Malgré la jeunesse du framework sa documentation parfois spartiate, le projet a de réelles forces et s'avère de plus en plus populaire dans le domaine. Bien que le développement de ce projet avance lentement, ce projet est aujourd'hui maintenu par des membres de l'équipe Guava.

Donc le benchmark que j'ai mis en place :

```java
public class PairCountsBenchmark extends SimpleBenchmark {

    @Param({ "1", "6", "25", "7", "111111111" }) private int K;  // différentes valeurs de la distance de la paire
    @Param({"10", "100", "1000", "10000"}) private int length;   // différentes tailles du tableau d'entier

    @Param private Distribution distribution;                    // Type de distribution SAWTOOTH, RANDOM, une autre

    private int[] values;

    @Override protected void setUp() throws Exception {          // Création du tableau d'entier
        values = distribution.create(length);
    }

    public int timePairCounts_V1(int repetition) {
        int dummy = 0;
        for (int i = 0; i < repetition; i++) {
            dummy += new PairCounts_V1().complementary_pairs(K, values);
        }
        return dummy;                                            // Variable modifiée dans la boucle pour forcer la JVM a ne pas supprimer cette boucle for.
    }

    public int timePairCounts_V2(int repetition) {
        int dummy = 0;
        for (int i = 0; i < repetition; i++) {
            dummy += new PairCounts_V2().complementary_pairs(K, values);
        }
        return dummy;
    }

    public enum Distribution {                                   // Le code de la distribution voulue
        SAWTOOTH {
            @Override
            int[] create(int length) {
                int[] result = new int[length];
                for (int i = 0; i < length; i += 5) {
                    result[i] = 0;
                    result[i + 1] = 1;
                    result[i + 2] = 2;
                    result[i + 3] = 3;
                    result[i + 4] = 4;
                }
                return result;
            }
        },
        RANDOM {
            @Override
            int[] create(int length) {
                Random random = new Random();
                int[] result = new int[length];
                for (int i = 0; i < length; i++) {
                    result[i] = random.nextInt();
                }
                return result;
            }
        };

        abstract int[] create(int length);
    }
}
```

## Dans le temps

Voilà il faut maintenant lancer le benchmark en ligne de commande. Et voici une partie de la sortie standard :

```text
 0% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=10} 117.03 ns; σ=0.53 ns @ 3 trials
 1% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=10} 410.92 ns; σ=3.93 ns @ 5 trials
 3% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=10} 118.95 ns; σ=0.54 ns @ 3 trials
 4% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=10} 346.11 ns; σ=1.04 ns @ 3 trials
 5% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=10} 124.75 ns; σ=0.42 ns @ 3 trials
 6% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=10} 343.76 ns; σ=0.60 ns @ 3 trials
 8% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=10} 116.55 ns; σ=0.11 ns @ 3 trials
 9% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=10} 340.24 ns; σ=0.88 ns @ 3 trials
10% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=10} 124.34 ns; σ=1.17 ns @ 8 trials
11% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=10} 375.23 ns; σ=2.22 ns @ 3 trials
13% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=10} 120.60 ns; σ=0.56 ns @ 3 trials
14% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=10} 399.89 ns; σ=6.66 ns @ 10 trials
15% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=10} 122.83 ns; σ=0.50 ns @ 3 trials
16% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=10} 405.20 ns; σ=6.81 ns @ 10 trials
18% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=10} 121.31 ns; σ=0.82 ns @ 3 trials
19% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=10} 406.40 ns; σ=7.90 ns @ 10 trials
20% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=10} 123.25 ns; σ=0.44 ns @ 3 trials
21% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=10} 403.80 ns; σ=4.77 ns @ 10 trials
23% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=10} 122.23 ns; σ=0.95 ns @ 3 trials
24% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=10} 401.21 ns; σ=9.94 ns @ 10 trials
25% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=100} 3554.64 ns; σ=13.22 ns @ 3 trials
26% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=100} 3353.02 ns; σ=28.50 ns @ 4 trials
28% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=100} 3616.36 ns; σ=23.17 ns @ 3 trials
29% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=100} 2759.35 ns; σ=7.49 ns @ 3 trials
30% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=100} 3230.40 ns; σ=6.89 ns @ 3 trials
31% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=100} 2630.14 ns; σ=11.76 ns @ 3 trials
33% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=100} 3470.91 ns; σ=21.03 ns @ 3 trials
34% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=100} 2703.77 ns; σ=2.87 ns @ 3 trials
35% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=100} 3237.29 ns; σ=31.98 ns @ 3 trials
36% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=100} 3462.79 ns; σ=14.26 ns @ 3 trials
38% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=100} 3664.38 ns; σ=22.84 ns @ 3 trials
39% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=100} 4782.95 ns; σ=21.09 ns @ 3 trials
40% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=100} 3666.78 ns; σ=11.56 ns @ 3 trials
41% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=100} 4829.99 ns; σ=18.42 ns @ 3 trials
43% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=100} 3669.01 ns; σ=3.86 ns @ 3 trials
44% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=100} 4800.20 ns; σ=27.45 ns @ 3 trials
45% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=100} 3697.72 ns; σ=16.69 ns @ 3 trials
46% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=100} 4867.33 ns; σ=39.87 ns @ 3 trials
48% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=100} 3663.00 ns; σ=18.92 ns @ 3 trials
49% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=100} 4792.09 ns; σ=23.93 ns @ 3 trials
50% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=1000} 253205.17 ns; σ=2409.40 ns @ 3 trials
51% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=1000} 28733.38 ns; σ=57.50 ns @ 3 trials
53% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=1000} 252201.82 ns; σ=1916.34 ns @ 3 trials
54% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=1000} 26003.20 ns; σ=79.01 ns @ 3 trials
55% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=1000} 254835.90 ns; σ=1142.59 ns @ 3 trials
56% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=1000} 24830.33 ns; σ=96.51 ns @ 3 trials
57% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=1000} 336495.11 ns; σ=3182.70 ns @ 3 trials
59% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=1000} 25425.31 ns; σ=236.34 ns @ 4 trials
60% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=1000} 254613.30 ns; σ=591.19 ns @ 3 trials
61% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=1000} 31176.21 ns; σ=34.10 ns @ 3 trials
63% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=1000} 264752.16 ns; σ=1911.17 ns @ 3 trials
64% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=1000} 53425.63 ns; σ=530.29 ns @ 5 trials
65% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=1000} 268841.98 ns; σ=1489.84 ns @ 3 trials
66% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=1000} 58635.35 ns; σ=387.69 ns @ 3 trials
68% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=1000} 266655.10 ns; σ=1012.10 ns @ 3 trials
69% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=1000} 59015.10 ns; σ=814.55 ns @ 10 trials
70% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=1000} 268504.45 ns; σ=482.22 ns @ 3 trials
71% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=1000} 58601.33 ns; σ=568.63 ns @ 3 trials
73% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=1000} 265022.05 ns; σ=1939.79 ns @ 3 trials
74% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=1000} 58487.85 ns; σ=587.15 ns @ 10 trials
75% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=10000} 23462428.57 ns; σ=79476.07 ns @ 3 trials
76% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=10000} 285100.65 ns; σ=3403.03 ns @ 10 trials
78% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=10000} 42566217.39 ns; σ=189344.42 ns @ 3 trials
79% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=10000} 273497.95 ns; σ=2738.46 ns @ 7 trials
80% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=10000} 24271754.10 ns; σ=74440.46 ns @ 3 trials
81% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=10000} 257143.69 ns; σ=1045.68 ns @ 3 trials
83% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=10000} 33062666.67 ns; σ=154823.32 ns @ 3 trials
84% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=10000} 270617.53 ns; σ=1484.45 ns @ 3 trials
85% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=10000} 24082112.90 ns; σ=67162.43 ns @ 3 trials
86% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=10000} 296935.84 ns; σ=197.58 ns @ 3 trials
88% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=10000} 24617137.50 ns; σ=222519.79 ns @ 4 trials
89% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=10000} 837504.04 ns; σ=7730.54 ns @ 4 trials
90% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=10000} 24380700.00 ns; σ=28021.86 ns @ 3 trials
91% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=10000} 861687.14 ns; σ=7389.43 ns @ 3 trials
93% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=10000} 24449950.00 ns; σ=137734.32 ns @ 3 trials
94% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=10000} 865862.78 ns; σ=6500.68 ns @ 3 trials
95% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=10000} 24474200.00 ns; σ=49916.49 ns @ 3 trials
96% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=10000} 860750.64 ns; σ=8304.85 ns @ 6 trials
98% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=10000} 24364925.00 ns; σ=72881.36 ns @ 3 trials
99% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=10000} 840977.96 ns; σ=8104.18 ns @ 10 trials

    benchmark length distribution         K       ns linear runtime
PairCounts_V1     10     SAWTOOTH         1      117 =
PairCounts_V1     10     SAWTOOTH         6      119 =
PairCounts_V1     10     SAWTOOTH        25      125 =
PairCounts_V1     10     SAWTOOTH         7      117 =
PairCounts_V1     10     SAWTOOTH 111111111      124 =
PairCounts_V1     10       RANDOM         1      121 =
PairCounts_V1     10       RANDOM         6      123 =
PairCounts_V1     10       RANDOM        25      121 =
PairCounts_V1     10       RANDOM         7      123 =
PairCounts_V1     10       RANDOM 111111111      122 =
PairCounts_V1    100     SAWTOOTH         1     3555 =
PairCounts_V1    100     SAWTOOTH         6     3616 =
PairCounts_V1    100     SAWTOOTH        25     3230 =
PairCounts_V1    100     SAWTOOTH         7     3471 =
PairCounts_V1    100     SAWTOOTH 111111111     3237 =
PairCounts_V1    100       RANDOM         1     3664 =
PairCounts_V1    100       RANDOM         6     3667 =
PairCounts_V1    100       RANDOM        25     3669 =
PairCounts_V1    100       RANDOM         7     3698 =
PairCounts_V1    100       RANDOM 111111111     3663 =
PairCounts_V1   1000     SAWTOOTH         1   253205 =
PairCounts_V1   1000     SAWTOOTH         6   252202 =
PairCounts_V1   1000     SAWTOOTH        25   254836 =
PairCounts_V1   1000     SAWTOOTH         7   336495 =
PairCounts_V1   1000     SAWTOOTH 111111111   254613 =
PairCounts_V1   1000       RANDOM         1   264752 =
PairCounts_V1   1000       RANDOM         6   268842 =
PairCounts_V1   1000       RANDOM        25   266655 =
PairCounts_V1   1000       RANDOM         7   268504 =
PairCounts_V1   1000       RANDOM 111111111   265022 =
PairCounts_V1  10000     SAWTOOTH         1 23462429 ================
PairCounts_V1  10000     SAWTOOTH         6 42566217 ==============================
PairCounts_V1  10000     SAWTOOTH        25 24271754 =================
PairCounts_V1  10000     SAWTOOTH         7 33062667 =======================
PairCounts_V1  10000     SAWTOOTH 111111111 24082113 ================
PairCounts_V1  10000       RANDOM         1 24617138 =================
PairCounts_V1  10000       RANDOM         6 24380700 =================
PairCounts_V1  10000       RANDOM        25 24449950 =================
PairCounts_V1  10000       RANDOM         7 24474200 =================
PairCounts_V1  10000       RANDOM 111111111 24364925 =================
PairCounts_V2     10     SAWTOOTH         1      411 =
PairCounts_V2     10     SAWTOOTH         6      346 =
PairCounts_V2     10     SAWTOOTH        25      344 =
PairCounts_V2     10     SAWTOOTH         7      340 =
PairCounts_V2     10     SAWTOOTH 111111111      375 =
PairCounts_V2     10       RANDOM         1      400 =
PairCounts_V2     10       RANDOM         6      405 =
PairCounts_V2     10       RANDOM        25      406 =
PairCounts_V2     10       RANDOM         7      404 =
PairCounts_V2     10       RANDOM 111111111      401 =
PairCounts_V2    100     SAWTOOTH         1     3353 =
PairCounts_V2    100     SAWTOOTH         6     2759 =
PairCounts_V2    100     SAWTOOTH        25     2630 =
PairCounts_V2    100     SAWTOOTH         7     2704 =
PairCounts_V2    100     SAWTOOTH 111111111     3463 =
PairCounts_V2    100       RANDOM         1     4783 =
PairCounts_V2    100       RANDOM         6     4830 =
PairCounts_V2    100       RANDOM        25     4800 =
PairCounts_V2    100       RANDOM         7     4867 =
PairCounts_V2    100       RANDOM 111111111     4792 =
PairCounts_V2   1000     SAWTOOTH         1    28733 =
PairCounts_V2   1000     SAWTOOTH         6    26003 =
PairCounts_V2   1000     SAWTOOTH        25    24830 =
PairCounts_V2   1000     SAWTOOTH         7    25425 =
PairCounts_V2   1000     SAWTOOTH 111111111    31176 =
PairCounts_V2   1000       RANDOM         1    53426 =
PairCounts_V2   1000       RANDOM         6    58635 =
PairCounts_V2   1000       RANDOM        25    59015 =
PairCounts_V2   1000       RANDOM         7    58601 =
PairCounts_V2   1000       RANDOM 111111111    58488 =
PairCounts_V2  10000     SAWTOOTH         1   285101 =
PairCounts_V2  10000     SAWTOOTH         6   273498 =
PairCounts_V2  10000     SAWTOOTH        25   257144 =
PairCounts_V2  10000     SAWTOOTH         7   270618 =
PairCounts_V2  10000     SAWTOOTH 111111111   296936 =
PairCounts_V2  10000       RANDOM         1   837504 =
PairCounts_V2  10000       RANDOM         6   861687 =
PairCounts_V2  10000       RANDOM        25   865863 =
PairCounts_V2  10000       RANDOM         7   860751 =
PairCounts_V2  10000       RANDOM 111111111   840978 =

vm: java
trial: 0

Process finished with exit code 0
```

Donc en fait Caliper créé une matrice multi-dimensionnelle des différents paramètres et pour chaque coordonnée dans cette matrice lance le test, bref un Scénario.

On voit dans les mesures faites par Caliper le temps pris par chaque méthode, l'écart type, et le nombre d'essai. Enfin dans une deuxième partie Caliper donne un synopsis des différents run et en particulier une colonne très intéressante '**linear time**'.

En observant le temps pris en fonction du nombre d'éléments pour chaque algo, on s'aperçoit que le temps pris par le premier algo augmente en effet par rapport au deuxième algo d'un facteur 5 qui augmente avec la taille du tableau. Bref on est loin d'un temps linéaire aka O(n).

Ce qui est aussi intéressant, c'est que le premier algo est plus efficace tant que le nombre d'élément dans le tableau d'entrée est inférieur à 100. Alors que la deuxième  qui utilise une structure plus élaboré ne montre des signes avantageux qu'à partir d'une centaine d'éléments. Ca me rappelle étrangement l'électronique ou les comportements des capacités et inductances changeant de nature lorsqu'on passe en haute fréquence.

## Dans l'espace

Alors pour faire les mesures des allocations, on peut aussi utiliser caliper, mais à l'heure de l'écriture de ce blog, il faut faire quelques petites choses en plus.

1. Caliper 0.5 RC1 vient avec le jar `java-allocation-instrumenter-2.0.jar` qui est sensé servir d'agent, cependant ce jar n'a pas été généré correctement pour servir d'agent. En fait il faut télécharger le jar `allocation.jar` de ce projet : [1 er février 2012](http://code.google.com/p/java-allocation-instrumenter/downloads/detail?name=allocation.jar&can=2&q=).
2. Avant de lancer le Runner en ligne de commande il faut ajouter la variable d'environnement `ALLOCATION_JAR`
  ```sh
  export ALLOCATION_JAR=/path/to/downloaded/allocation.jar
  ```
3. Enfin il est possible de lancer le même benchmark avec des tests sur la mémoire :
  ```java
  java -classpath $THE_CLASSPATH --measureMemory com.google.caliper.Runner PairCountsBenchmark
  ```

> A noter : Ne pas renommer `allocation.jar` en autre chose, sans ça vous n'aurez pas d'instrumentation !

Ce qui donne le résultat suivant, quasiment la même chose, mais avec les infos sur les allocations mémoire.

```text
 0% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=10} 119.06 ns; σ=1.08 ns @ 5 trials, allocated 1 instances for a total of 16B
 1% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=10} 468.80 ns; σ=4.56 ns @ 3 trials, allocated 9 instances for a total of 320B
 3% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=10} 115.83 ns; σ=0.46 ns @ 3 trials, allocated 1 instances for a total of 16B
 4% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=10} 426.45 ns; σ=3.65 ns @ 3 trials, allocated 9 instances for a total of 320B
 5% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=10} 125.06 ns; σ=0.52 ns @ 3 trials, allocated 1 instances for a total of 16B
 6% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=10} 407.38 ns; σ=1.42 ns @ 3 trials, allocated 9 instances for a total of 320B
 8% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=10} 116.38 ns; σ=1.11 ns @ 6 trials, allocated 1 instances for a total of 16B
 9% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=10} 425.28 ns; σ=0.46 ns @ 3 trials, allocated 9 instances for a total of 320B
10% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=10} 125.12 ns; σ=0.18 ns @ 3 trials, allocated 1 instances for a total of 16B
11% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=10} 439.33 ns; σ=3.29 ns @ 3 trials, allocated 19 instances for a total of 560B
13% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=10} 121.67 ns; σ=0.46 ns @ 3 trials, allocated 1 instances for a total of 16B
14% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=10} 482.23 ns; σ=3.74 ns @ 3 trials, allocated 34 instances for a total of 960B
15% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=10} 120.69 ns; σ=0.71 ns @ 3 trials, allocated 1 instances for a total of 16B
16% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=10} 455.67 ns; σ=3.55 ns @ 3 trials, allocated 34 instances for a total of 960B
18% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=10} 121.64 ns; σ=0.42 ns @ 3 trials, allocated 1 instances for a total of 16B
19% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=10} 464.34 ns; σ=7.45 ns @ 10 trials, allocated 34 instances for a total of 960B
20% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=10} 120.85 ns; σ=0.38 ns @ 3 trials, allocated 1 instances for a total of 16B
21% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=10} 454.11 ns; σ=5.22 ns @ 10 trials, allocated 34 instances for a total of 960B
23% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=10} 122.14 ns; σ=1.12 ns @ 4 trials, allocated 1 instances for a total of 16B
24% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=10} 481.91 ns; σ=7.18 ns @ 10 trials, allocated 34 instances for a total of 960B
25% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=100} 3530.68 ns; σ=18.40 ns @ 3 trials, allocated 1 instances for a total of 16B
26% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=100} 3095.60 ns; σ=7.39 ns @ 3 trials, allocated 9 instances for a total of 320B
28% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=100} 3677.07 ns; σ=36.07 ns @ 3 trials, allocated 1 instances for a total of 16B
29% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=100} 2871.70 ns; σ=96.87 ns @ 10 trials, allocated 9 instances for a total of 320B
30% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=100} 3253.18 ns; σ=16.46 ns @ 3 trials, allocated 1 instances for a total of 16B
31% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=100} 2830.61 ns; σ=61.72 ns @ 10 trials, allocated 9 instances for a total of 320B
33% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=100} 3868.67 ns; σ=36.13 ns @ 3 trials, allocated 1 instances for a total of 16B
34% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=100} 2747.84 ns; σ=26.29 ns @ 3 trials, allocated 9 instances for a total of 320B
35% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=100} 3240.13 ns; σ=12.72 ns @ 3 trials, allocated 1 instances for a total of 16B
36% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=100} 2941.78 ns; σ=39.47 ns @ 10 trials, allocated 109 instances for a total of 2720B
38% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=100} 3703.12 ns; σ=9.85 ns @ 3 trials, allocated 1 instances for a total of 16B
39% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=100} 4598.91 ns; σ=18.33 ns @ 3 trials, allocated 308 instances for a total of 10144B
40% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=100} 3709.97 ns; σ=12.28 ns @ 3 trials, allocated 1 instances for a total of 16B
41% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=100} 4520.00 ns; σ=27.93 ns @ 3 trials, allocated 308 instances for a total of 10144B
43% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=100} 3665.28 ns; σ=13.51 ns @ 3 trials, allocated 1 instances for a total of 16B
44% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=100} 4527.80 ns; σ=35.77 ns @ 3 trials, allocated 308 instances for a total of 10144B
45% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=100} 3701.99 ns; σ=22.06 ns @ 3 trials, allocated 1 instances for a total of 16B
46% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=100} 4498.57 ns; σ=45.15 ns @ 3 trials, allocated 308 instances for a total of 10144B
48% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=100} 3729.27 ns; σ=16.84 ns @ 3 trials, allocated 1 instances for a total of 16B
49% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=100} 4579.87 ns; σ=8.15 ns @ 3 trials, allocated 308 instances for a total of 10144B
50% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=1000} 254954.05 ns; σ=648.33 ns @ 3 trials, allocated 1 instances for a total of 16B
51% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=1000} 29495.00 ns; σ=202.71 ns @ 3 trials, allocated 374 instances for a total of 6160B
53% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=1000} 272123.84 ns; σ=1947.52 ns @ 3 trials, allocated 1 instances for a total of 16B
54% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=1000} 26549.46 ns; σ=169.74 ns @ 3 trials, allocated 374 instances for a total of 6160B
55% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=1000} 258246.52 ns; σ=1838.14 ns @ 3 trials, allocated 1 instances for a total of 16B
56% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=1000} 25816.29 ns; σ=238.66 ns @ 3 trials, allocated 374 instances for a total of 6160B
57% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=1000} 340760.25 ns; σ=714.81 ns @ 3 trials, allocated 1 instances for a total of 16B
59% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=1000} 26070.26 ns; σ=186.94 ns @ 3 trials, allocated 374 instances for a total of 6160B
60% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=1000} 255782.03 ns; σ=388.87 ns @ 3 trials, allocated 1 instances for a total of 16B
61% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=1000} 26950.40 ns; σ=491.82 ns @ 10 trials, allocated 1374 instances for a total of 30160B
63% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=1000} 268871.49 ns; σ=2890.36 ns @ 10 trials, allocated 1 instances for a total of 16B
64% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=1000} 50888.14 ns; σ=106.59 ns @ 3 trials, allocated 3011 instances for a total of 96528B
65% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=1000} 268566.99 ns; σ=2398.41 ns @ 4 trials, allocated 1 instances for a total of 16B
66% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=1000} 63228.83 ns; σ=495.26 ns @ 3 trials, allocated 3011 instances for a total of 96528B
68% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=1000} 265538.89 ns; σ=323.95 ns @ 3 trials, allocated 1 instances for a total of 16B
69% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=1000} 51922.10 ns; σ=491.28 ns @ 3 trials, allocated 3011 instances for a total of 96528B
70% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=1000} 267048.78 ns; σ=1934.21 ns @ 3 trials, allocated 1 instances for a total of 16B
71% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=1000} 54193.80 ns; σ=526.83 ns @ 3 trials, allocated 3011 instances for a total of 96528B
73% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=1000} 270239.69 ns; σ=1291.59 ns @ 3 trials, allocated 1 instances for a total of 16B
74% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=1000} 51265.07 ns; σ=547.18 ns @ 10 trials, allocated 3011 instances for a total of 96528B
75% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=SAWTOOTH, length=10000} 23672536.59 ns; σ=131459.47 ns @ 3 trials, allocated 1 instances for a total of 16B
76% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=SAWTOOTH, length=10000} 297356.00 ns; σ=1073.95 ns @ 3 trials, allocated 9374 instances for a total of 150160B
78% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=SAWTOOTH, length=10000} 42248173.91 ns; σ=364119.70 ns @ 3 trials, allocated 1 instances for a total of 16B
79% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=SAWTOOTH, length=10000} 273661.54 ns; σ=1544.97 ns @ 3 trials, allocated 9374 instances for a total of 150160B
80% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=SAWTOOTH, length=10000} 24378375.00 ns; σ=184303.85 ns @ 3 trials, allocated 1 instances for a total of 16B
81% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=SAWTOOTH, length=10000} 269039.83 ns; σ=123.09 ns @ 3 trials, allocated 9374 instances for a total of 150160B
83% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=SAWTOOTH, length=10000} 33378571.43 ns; σ=154321.88 ns @ 3 trials, allocated 1 instances for a total of 16B
84% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=SAWTOOTH, length=10000} 266848.28 ns; σ=1354.38 ns @ 3 trials, allocated 9374 instances for a total of 150160B
85% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=SAWTOOTH, length=10000} 24348450.00 ns; σ=158682.88 ns @ 3 trials, allocated 1 instances for a total of 16B
86% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=SAWTOOTH, length=10000} 281623.08 ns; σ=257.89 ns @ 3 trials, allocated 19374 instances for a total of 390160B
88% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=1, distribution=RANDOM, length=10000} 24604800.00 ns; σ=84624.92 ns @ 3 trials, allocated 1 instances for a total of 16B
89% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=1, distribution=RANDOM, length=10000} 826805.04 ns; σ=8011.71 ns @ 7 trials, allocated 30014 instances for a total of 931264B
90% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=6, distribution=RANDOM, length=10000} 24467945.29 ns; σ=230301.47 ns @ 4 trials, allocated 1 instances for a total of 16B
91% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=6, distribution=RANDOM, length=10000} 822419.69 ns; σ=8032.73 ns @ 9 trials, allocated 30014 instances for a total of 931264B
93% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=25, distribution=RANDOM, length=10000} 24666650.00 ns; σ=65345.42 ns @ 3 trials, allocated 1 instances for a total of 16B
94% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=25, distribution=RANDOM, length=10000} 819003.80 ns; σ=7662.68 ns @ 3 trials, allocated 30014 instances for a total of 931264B
95% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=7, distribution=RANDOM, length=10000} 24432650.00 ns; σ=210910.19 ns @ 3 trials, allocated 1 instances for a total of 16B
96% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=7, distribution=RANDOM, length=10000} 835849.98 ns; σ=9613.89 ns @ 10 trials, allocated 30014 instances for a total of 931264B
98% Scenario{vm=java, trial=0, benchmark=PairCounts_V1, K=111111111, distribution=RANDOM, length=10000} 24464100.00 ns; σ=41413.99 ns @ 3 trials, allocated 1 instances for a total of 16B
99% Scenario{vm=java, trial=0, benchmark=PairCounts_V2, K=111111111, distribution=RANDOM, length=10000} 835717.46 ns; σ=24729.21 ns @ 10 trials, allocated 30013 instances for a total of 931264B

 benchmark length distribution         K instances        B       ns linear runtime
PairCounts_V1     10     SAWTOOTH         1     1.000     16.0      119 =
PairCounts_V1     10     SAWTOOTH         6     1.000     16.0      116 =
PairCounts_V1     10     SAWTOOTH        25     1.000     16.0      125 =
PairCounts_V1     10     SAWTOOTH         7     1.000     16.0      116 =
PairCounts_V1     10     SAWTOOTH 111111111     1.000     16.0      125 =
PairCounts_V1     10       RANDOM         1     1.000     16.0      122 =
PairCounts_V1     10       RANDOM         6     1.000     16.0      121 =
PairCounts_V1     10       RANDOM        25     1.000     16.0      122 =
PairCounts_V1     10       RANDOM         7     1.000     16.0      121 =
PairCounts_V1     10       RANDOM 111111111     1.000     16.0      122 =
PairCounts_V1    100     SAWTOOTH         1     1.000     16.0     3531 =
PairCounts_V1    100     SAWTOOTH         6     1.000     16.0     3677 =
PairCounts_V1    100     SAWTOOTH        25     1.000     16.0     3253 =
PairCounts_V1    100     SAWTOOTH         7     1.000     16.0     3869 =
PairCounts_V1    100     SAWTOOTH 111111111     1.000     16.0     3240 =
PairCounts_V1    100       RANDOM         1     1.000     16.0     3703 =
PairCounts_V1    100       RANDOM         6     1.000     16.0     3710 =
PairCounts_V1    100       RANDOM        25     1.000     16.0     3665 =
PairCounts_V1    100       RANDOM         7     1.000     16.0     3702 =
PairCounts_V1    100       RANDOM 111111111     1.000     16.0     3729 =
PairCounts_V1   1000     SAWTOOTH         1     1.000     16.0   254954 =
PairCounts_V1   1000     SAWTOOTH         6     1.000     16.0   272124 =
PairCounts_V1   1000     SAWTOOTH        25     1.000     16.0   258247 =
PairCounts_V1   1000     SAWTOOTH         7     1.000     16.0   340760 =
PairCounts_V1   1000     SAWTOOTH 111111111     1.000     16.0   255782 =
PairCounts_V1   1000       RANDOM         1     1.000     16.0   268871 =
PairCounts_V1   1000       RANDOM         6     1.000     16.0   268567 =
PairCounts_V1   1000       RANDOM        25     1.000     16.0   265539 =
PairCounts_V1   1000       RANDOM         7     1.000     16.0   267049 =
PairCounts_V1   1000       RANDOM 111111111     1.000     16.0   270240 =
PairCounts_V1  10000     SAWTOOTH         1     1.000     16.0 23672537 ================
PairCounts_V1  10000     SAWTOOTH         6     1.000     16.0 42248174 ==============================
PairCounts_V1  10000     SAWTOOTH        25     1.000     16.0 24378375 =================
PairCounts_V1  10000     SAWTOOTH         7     1.000     16.0 33378571 =======================
PairCounts_V1  10000     SAWTOOTH 111111111     1.000     16.0 24348450 =================
PairCounts_V1  10000       RANDOM         1     1.000     16.0 24604800 =================
PairCounts_V1  10000       RANDOM         6     1.000     16.0 24467945 =================
PairCounts_V1  10000       RANDOM        25     1.000     16.0 24666650 =================
PairCounts_V1  10000       RANDOM         7     1.000     16.0 24432650 =================
PairCounts_V1  10000       RANDOM 111111111     1.000     16.0 24464100 =================
PairCounts_V2     10     SAWTOOTH         1     9.000    320.0      469 =
PairCounts_V2     10     SAWTOOTH         6     9.000    320.0      426 =
PairCounts_V2     10     SAWTOOTH        25     9.000    320.0      407 =
PairCounts_V2     10     SAWTOOTH         7     9.000    320.0      425 =
PairCounts_V2     10     SAWTOOTH 111111111    19.000    560.0      439 =
PairCounts_V2     10       RANDOM         1    34.000    960.0      482 =
PairCounts_V2     10       RANDOM         6    34.000    960.0      456 =
PairCounts_V2     10       RANDOM        25    34.000    960.0      464 =
PairCounts_V2     10       RANDOM         7    34.000    960.0      454 =
PairCounts_V2     10       RANDOM 111111111    34.000    960.0      482 =
PairCounts_V2    100     SAWTOOTH         1     9.000    320.0     3096 =
PairCounts_V2    100     SAWTOOTH         6     9.000    320.0     2872 =
PairCounts_V2    100     SAWTOOTH        25     9.000    320.0     2831 =
PairCounts_V2    100     SAWTOOTH         7     9.000    320.0     2748 =
PairCounts_V2    100     SAWTOOTH 111111111   109.000   2720.0     2942 =
PairCounts_V2    100       RANDOM         1   308.000  10144.0     4599 =
PairCounts_V2    100       RANDOM         6   308.000  10144.0     4520 =
PairCounts_V2    100       RANDOM        25   308.000  10144.0     4528 =
PairCounts_V2    100       RANDOM         7   308.000  10144.0     4499 =
PairCounts_V2    100       RANDOM 111111111   308.000  10144.0     4580 =
PairCounts_V2   1000     SAWTOOTH         1   374.000   6160.0    29495 =
PairCounts_V2   1000     SAWTOOTH         6   374.000   6160.0    26549 =
PairCounts_V2   1000     SAWTOOTH        25   374.000   6160.0    25816 =
PairCounts_V2   1000     SAWTOOTH         7   374.000   6160.0    26070 =
PairCounts_V2   1000     SAWTOOTH 111111111  1374.000  30160.0    26950 =
PairCounts_V2   1000       RANDOM         1  3011.000  96528.0    50888 =
PairCounts_V2   1000       RANDOM         6  3011.000  96528.0    63229 =
PairCounts_V2   1000       RANDOM        25  3011.000  96528.0    51922 =
PairCounts_V2   1000       RANDOM         7  3011.000  96528.0    54194 =
PairCounts_V2   1000       RANDOM 111111111  3011.000  96528.0    51265 =
PairCounts_V2  10000     SAWTOOTH         1  9374.000 150160.0   297356 =
PairCounts_V2  10000     SAWTOOTH         6  9374.000 150160.0   273662 =
PairCounts_V2  10000     SAWTOOTH        25  9374.000 150160.0   269040 =
PairCounts_V2  10000     SAWTOOTH         7  9374.000 150160.0   266848 =
PairCounts_V2  10000     SAWTOOTH 111111111 19374.000 390160.0   281623 =
PairCounts_V2  10000       RANDOM         1 30014.000 931264.0   826805 =
PairCounts_V2  10000       RANDOM         6 30014.000 931264.0   822420 =
PairCounts_V2  10000       RANDOM        25 30014.000 931264.0   819004 =
PairCounts_V2  10000       RANDOM         7 30014.000 931264.0   835850 =
PairCounts_V2  10000       RANDOM 111111111 30013.000 931264.0   835717 =

vm: java
trial: 0

Process finished with exit code 0
```

C'est effectivement une information utile, la première version de l'algo, ne fait en fait qu'une seule allocation de 16B (donc en fait un seul objet), c'est à dire peanuts, on est dans du O(1) en complexité spatiale. La deuxième qui est notamment basée sur une HashMap alloue nettement plus d'objets, mais est définitivement plus rapide, on a on ici une complexité spatiale de O(n).

Comme quoi il y a potentiellement des compromis à faire dans le choix d'un algo, la rapidité ou la faible consommation mémoire peuvent venir avec un coût dans une autre dimension.

# Pour conclure

* Premier point, il faut absolument être au point pour des tests d'embauche plus sur le sujet, même si je trouve limité ces tests dans leur capacité à identifier ou filtrer les bons développeurs (l'algorithmie n'est pas certainement pas le seul critère d'un individu doué), c'est toujours bien de pouvoir les reéussir !
* Caliper offre un outillage plutôt facile d'utilisation pour faire des microbenchmark, dans la limite de validité d'un microbenchmark, il y a plein de mise en garde sur le sujet, sur le site de Caliper [[4](http://www.azulsystems.com/events/javaone_2009/session/2009_J1_Benchmark.pdf)].
* Quand on écrit du code c'est une bonne idée de penser à la complexité temporelle et spatiale, dont la notation pour les deux est le grand O. Caliper pourrait d'ailleurs se voir ajouter le moyen de mesurer la consommation mémoire (en lien avec la complexité spatiale). Évidement ce que je dis ne veut pas dire optimiser prématurément, mais juste de réfléchir à quel type de performance on peut s'attendre pour telle ou telle partie du code, et éventuellement de porter plus tard un effort spécifique.

A noter que Caliper offre d'autres possibilités de mesurer la performance à travers une méthode annotée par `@ArbitraryMeasurement`.
