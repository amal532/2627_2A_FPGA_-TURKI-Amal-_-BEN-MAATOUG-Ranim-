**TP1--FPGA--Tutoriel Quartus**
**Auteurs: Ranim ben maatoug et Amal Turki**

**Introduction**
Ce TP a pour but de découvrir le logiciel Quartus Prime et de tester des composants VHDL sur une carte FPGA DE10-Nano (puce 5CSEBA6U23I7). Nous avons d'abord créé un projet, assigné les broches avec le Pin Planner, puis compilé et programmé la carte avec un composant simple (bouton vers LED). Nous sommes ensuite passés à la logique séquentielle : une LED qui clignote grâce à une bascule et à un compteur qui ralentit l'horloge de 50 MHz, avec un reset actif à l'état bas. Enfin, nous avons conçu un chenillard sur 10 LED. Pour chaque composant, nous avons comparé notre schéma avec celui généré par le RTL Viewer.

**Objectifs:**
Parcourir toute la chaîne VHDL → simulation → synthèse → placement/routage → bitstream → test sur carte, passer du combinatoire au séquentiel, puis concevoir un système complet (encodeurs, contrôleur HDMI, mémoire).

**1.Prise en main de Quartus**
On a commencé par l'alimentation externe de la carte à travers le port USB, puis la création du projet qui doit avoir le meme nom que l'entity.
/n
**1.1 Premier composant VHDL**

**code source** /n
library ieee;
use ieee.std_logic_1164.all;

entity tuto_fpga is
    port (
        pushl : in std_logic;
        led0 : out std_logic
    );
end entity tuto_fpga;

architecture rtl of tuto_fpga is
begin
    led0 <= pushl;
end architecture rtl;
/br
**=>La synthèse doit être lancée avant le Pin Planner : c'est elle qui fait connaître à Quartus les noms des ports de l'entity.**


| Signal | Broche   | Direction |
|--------|----------|-----------|
| led0   | PIN_AG28 | sortie    |
| pushl  | PIN_AH27 | entrée    |

**photo pin planner**

<img width="1590" height="1005" alt="8a173fa0-1447-46c0-b60e-0fecee3582fc" src="https://github.com/user-attachments/assets/658c8b7a-1d25-4a4d-83a8-c5754729e800" />


**1.2 Compilation et programmation de la carte**
**Compile Design** enchaîne : analyse et synthèse, fitter (placement et routage), assembleur (génération du .sof), analyse de timing.

**1.3 Comportement inversé**
Le comportement est inversé! La LED est allumée par défaut et s'éteind lorsque l'on appuie sur l'encodeur.
Donc one doit inverser le comportement par changement de la partie architecture:
architecture rtl of tuto_fpga is
begin
    led0 <=**not** pushl;
end architecture rtl;


**2.Faire clignoter une LED**
**Question 1**
<img width="997" height="201" alt="c7006007-44c3-489d-9917-5a096b374d31" src="https://github.com/user-attachments/assets/08d2a4ec-3336-4aac-b1a5-b700bef787af" />

**code source pour faire clignoter le LED**
library ieee;
use ieee.std_logic_1164.all;

entity led_blink is
    port (
        i_clk : in std_logic;
        i_rst_n : in std_logic;
        o_led : out std_logic
    );
end entity led_blink;

architecture rtl of led_blink is
    signal r_led : std_logic := '0';
begin
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_led <= '0';
        elsif (rising_edge(i_clk)) then
            r_led <= not r_led;
        end if;
    end process;
    o_led <= r_led;
end architecture rtl;

**Explication du code**
Ce code décrit un composant nommé led_blink qui fait changer l'état d'une LED à chaque front montant de l'horloge : la bibliothèque ieee fournit le type std_logic, l'entité définit l'interface avec une horloge i_clk, un reset i_rst_n actif à l'état bas (d'où le _n) et une sortie o_led, puis l'architecture déclare un signal interne r_led qui mémorise l'état de la LED (valeur initiale '0'). Le process, sensible à i_clk et i_rst_n, remet r_led à '0' immédiatement dès que le reset vaut '0' (reset asynchrone, testé avant l'horloge), sinon, à chaque front montant de l'horloge, il inverse r_led avec not r_led, et l'instruction o_led <= r_led relie simplement la sortie à ce registre. En matériel, cela correspond à une bascule D avec reset asynchrone dont l'entrée est reliée à sa propre sortie inversée, mais comme l'horloge est à 50 MHz, la LED clignoterait à 25 MHz, trop vite pour être vue, d'où la nécessité d'ajouter ensuite un compteur pour ralentir le clignotement.

**Schéma correspondant à ce code VHDL**
<img width="902" height="435" alt="2588293b-b87f-453b-99eb-c168dc71535e" src="https://github.com/user-attachments/assets/fb1691fa-f159-4090-96f9-3fa059c37842" />

**Le schéma proposé par RTL Viewer**
<img width="1007" height="777" alt="c718cb8e-0a49-4acb-81a3-181bd312e82f" src="https://github.com/user-attachments/assets/fd0835d5-2461-4058-bb0c-c32ed7a67549" />

**Comparaison entre les deux schemas**
Le schéma proposé par Quartus (RTL Viewer) correspond à celui tracé à partir du code VHDL. Il montre un registre r_led cadencé par i_clk sur front montant, avec un reset asynchrone actif à l'état bas (i_rst_n sur CLRN). La sortie Q est renvoyée sur l'entrée D après inversion (représentée par un rond sur l'entrée D), et reliée à la sortie o_led. La seule différence est que Quartus intègre l'inverseur à l'entrée D du registre et fait apparaître une entrée de reset synchrone SCLR non utilisée (forcée à 0). À chaque front montant de i_clk, r_led change d'état : la fréquence de la LED est donc celle de l'horloge divisée par 2.


**Modification de process**
Lorsque on a modifié le process qui permet de diviser la fréquence, le code a fait des erreurs de compilation car le r_led_enable était utilisé sans être déclaré et aucun process n'écrivait dans r_led, qui restait à '0' : la LED restait éteinte même si le compteur tournait. Donc pour le faire marcher on a du faire des modifications au code.
**Nouveau code**
library ieee;
use ieee.std_logic_1164.all;

entity led_blink is
    port (
        i_clk   : in  std_logic;
        i_rst_n : in  std_logic;
        o_led   : out std_logic
    );
end entity led_blink;

architecture rtl of led_blink is
    signal r_led        : std_logic := '0';
    signal r_led_enable : std_logic := '0';
begin
    -- Diviseur de fréquence : une impulsion toutes les ~100 ms
    process(i_clk, i_rst_n)
        variable counter : natural range 0 to 5000000 := 0;
    begin
        if (i_rst_n = '0') then
            counter := 0;
            r_led_enable <= '0';
        elsif (rising_edge(i_clk)) then
            if (counter = 5000000) then
                counter := 0;
                r_led_enable <= '1';
            else
                counter := counter + 1;
                r_led_enable <= '0';
            end if;
        end if;
    end process;

    -- Bascule de la LED sur autorisation
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_led <= '0';
        elsif (rising_edge(i_clk)) then
            if (r_led_enable = '1') then
                r_led <= not r_led;
            end if;
        end if;
    end process;

    o_led <= r_led;
end architecture rtl;
****
**Explication du code**
Ce code fait clignoter la LED à environ 5 Hz grâce à deux processus synchrones qui travaillent en parallèle sur la seule horloge de 50 MHz : le premier est un diviseur de fréquence dont la variable counter (bornée à 5 000 000) s'incrémente à chaque front montant, et lorsqu'elle atteint sa valeur maximale elle repart à zéro pendant que le registre r_led_enable passe à '1' pendant un seul cycle, ce qui produit une impulsion de 20 ns toutes les 5 000 001 périodes, soit environ 100 ms ; le second process inverse r_led (r_led <= not r_led) uniquement quand cette impulsion vaut '1', la LED change donc d'état toutes les 100 ms, ce qui donne une période d'environ 200 ms, et o_led <= r_led relie simplement la sortie à ce registre, tandis que le reset asynchrone actif à l'état bas i_rst_n remet le compteur, l'enable et la LED à zéro pour garantir un état initial connu. Ce choix d'un signal d'autorisation (enable) plutôt que d'une horloge dérivée est une bonne pratique, car le design reste entièrement synchrone sur une seule horloge, ce qui évite les problèmes de skew, de traversée de domaines d'horloge et simplifie l'analyse de timing ; en matériel, on retrouve dans le RTL Viewer un compteur avec comparateur, une bascule D produisant l'enable et une bascule D avec enable pour la LED, et la valeur 5 000 000 fixe la fréquence de clignotement, elle pourrait d'ailleurs être rendue configurable avec un generic.

**Schéma proposé correspondant au nouveau code**
<img width="923" height="550" alt="ab5dbc3e-f8bc-45f0-8798-25e0a16d9661" src="https://github.com/user-attachments/assets/03e7d00a-f3e3-4e3c-82c1-c5934976d212" />

**Schéma généré par RTL Viewer**
<img width="1816" height="1136" alt="93ac9317-18dc-4d96-aae9-7916a778831c" src="https://github.com/user-attachments/assets/4653f94a-0cc0-4d83-b1f4-168e2d49e5ca" />

**Comparaison entre les deux schémas**
Le schéma généré par Quartus (RTL Viewer) correspond à celui que j'avais tracé à partir du code VHDL. On y retrouve le compteur de 23 bits, le comparateur avec la valeur 5 000 000 (écrite 4C4B40 en hexadécimal), les registres r_led_enable et r_led, et la sortie o_led.

Il y a seulement quelques différences de représentation. Pour compter, Quartus utilise un additionneur (+1) suivi d'un multiplexeur : quand le comparateur détecte 5 000 000, le multiplexeur remet le compteur à 0, sinon il laisse passer compteur + 1. Pour la LED, Quartus utilise l'entrée d'autorisation ENA du registre r_led : la LED ne change d'état que lorsque r_led_enable vaut 1, et l'inversion (not r_led) est faite sur l'entrée D. Enfin, les entrées SCLR (reset synchrone) sont bloquées à 0 et ne servent pas : le reset utilisé est celui de CLRN, relié à i_rst_n, qui est asynchrone et actif à l'état bas.

Le circuit synthétisé est donc bien conforme au code VHDL : un compteur qui génère une impulsion toutes les 0,1 seconde environ, et qui fait changer la LED d'état à chaque impulsion.

Question 11 : que veut dire _n ?

Le _n signifie « actif à l'état bas » : le reset se déclenche quand le signal vaut '0', et non '1'. C'est ce qu'on voit dans le code avec if (i_rst_n = '0') then.

La raison : le bouton KEY0 vaut '1' quand on n'appuie pas, et '0' quand on appuie. Avec un reset actif à l'état bas, appuyer sur le bouton déclenche le reset, et le circuit fonctionne normalement quand on n'appuie pas.

**Chenillard**

**code**
library ieee;
use ieee.std_logic_1164.all;

entity chenillard is
    port (
        i_clk   : in  std_logic;
        i_rst_n : in  std_logic;
        o_led   : out std_logic_vector(9 downto 0)
    );
end entity chenillard;

architecture rtl of chenillard is
    signal r_led        : std_logic_vector(9 downto 0) := "0000000001";
    signal r_led_enable : std_logic := '0';
begin
    -- Diviseur de fréquence : une impulsion toutes les ~100 ms
    process(i_clk, i_rst_n)
        variable counter : natural range 0 to 5000000 := 0;
    begin
        if (i_rst_n = '0') then
            counter := 0;
            r_led_enable <= '0';
        elsif (rising_edge(i_clk)) then
            if (counter = 5000000) then
                counter := 0;
                r_led_enable <= '1';
            else
                counter := counter + 1;
                r_led_enable <= '0';
            end if;
        end if;
    end process;

    -- Décalage circulaire de la LED allumée sur autorisation
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then
            r_led <= "0000000001";
        elsif (rising_edge(i_clk)) then
            if (r_led_enable = '1') then
                r_led <= r_led(8 downto 0) & r_led(9);
            end if;
        end if;
    end process;

    o_led <= r_led;
end architecture rtl;
**Explication du code**
e code décrit un chenillard de 10 LED en VHDL. Après l'import de std_logic_1164 (qui fournit les types std_logic et std_logic_vector), l'entité définit les broches : l'horloge i_clk (50 MHz), le reset actif à l'état bas i_rst_n et les 10 sorties o_led. Dans l'architecture, r_led mémorise l'état des LED (une seule allumée au départ, "0000000001") et r_led_enable est une impulsion d'un cycle d'horloge. Le premier process est un diviseur de fréquence : un compteur s'incrémente à chaque front montant de l'horloge et, en atteignant 5 000 000 (soit environ 100 ms à 50 MHz), il se remet à zéro et lève r_led_enable à '1' pendant un cycle. Le second process réagit à cette impulsion : à chaque fois qu'elle arrive, il décale r_led d'un cran vers la gauche grâce à la concaténation r_led(8 downto 0) & r_led(9), et le bit sorti par le haut revient en bas, ce qui donne un mouvement circulaire de la LED allumée. Les deux process ont un reset asynchrone qui remet le compteur à 0 et la LED 0 allumée, et o_led <= r_led relie simplement les registres aux sorties. On utilise ainsi un signal d'enable plutôt qu'une horloge divisée, ce qui garde tout le design synchrone sur la seule horloge 50 MHz, et la vitesse se règle en changeant la valeur 5 000 000.
**Vidéo Démonstrative**
https://github.com/user-attachments/assets/39c9006a-f1c8-4eef-8e78-5c240f43b60d

**RTL Viewer**
<img width="1823" height="1126" alt="4e9a9ae6-ea2e-45b6-bd9d-a671ea8a8995" src="https://github.com/user-attachments/assets/08cb9e3f-05b2-45d6-a0b8-5f4c719fca61" />

**Conclusion**
Ce TP nous a permis de suivre toutes les étapes de conception sur FPGA : écriture du VHDL, synthèse, assignation des broches, compilation et programmation de la carte. Nous avons compris qu'un code VHDL décrit du matériel (registres, compteurs, comparateurs) et non un programme, et que le RTL Viewer permet de vérifier que le circuit synthétisé correspond bien à ce que l'on a prévu. Nous avons aussi retenu l'importance de ralentir l'horloge avec un compteur pour obtenir un clignotement visible, et d'utiliser un reset sur chaque registre. Enfin, le chenillard nous a permis de réutiliser ces notions dans un composant que nous avons conçu nous-mêmes.



















