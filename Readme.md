# Git-alapú dinamikus Ansible role kezelés Galaxy szerver nélkül

Ez a minta egy gyakorlati megoldást mutat arra, hogyan lehet Ansible role-okat közvetlenül Git repository-kból kezelni és letölteni, dedikált Ansible Galaxy szerver üzemeltetése nélkül.

Az alapötlet egyszerű: a role-ok futás közben töltődnek le Gitből, a hozzáférési adatok dinamikusan kerülnek behelyettesítésre, a role verziók pedig Git tagre vagy commit SHA-ra vannak rögzítve a reprodukálható futás érdekében.

A cél az, hogy a role dependency-k ne a role-ok belsejébe legyenek elrejtve, hanem explicit módon, projekt vagy playbook szinten legyenek kezelve.

## 📁 Ajánlott könyvtárstruktúra

A projekt önállósága és átláthatósága érdekében érdemes egyszerű, lapos könyvtárstruktúrát használni.

A `roles/` könyvtár automatikusan létrejön és feltöltődik a futás során.

```text
my-ansible-project/
├── requirements.yml.j2    # Jinja2 sablon a Git hitelesítéshez
├── playbook.yml           # Fő playbook
└── roles/                 # Ide töltődnek le automatikusan a role-ok
```

---

## 📄 Forrásfájlok

### 1. `requirements.yml.j2`

Ez a sablon definiálja a szükséges role-okat, és futás közben behelyettesíti a Git hozzáférési adatokat.

Éles környezetben érdemes védett Git tageket vagy commit SHA-kat használni. A tag olvashatóbb, a commit SHA viszont erősebb reprodukálhatóságot ad.

```yaml
---
roles:
  - name: company.nginx
    src: "git+https://{{ git_user | urlencode }}:{{ git_token | urlencode }}@git.example.com/ansible-roles/nginx.git"
    version: "v3.1.0"

  - name: company.postgresql
    src: "git+https://{{ git_user | urlencode }}:{{ git_token | urlencode }}@git.example.com/ansible-roles/postgresql.git"
    version: "v3.4.0"
```

---

### 2. `playbook.yml`

Ez a playbook vezérli a teljes folyamatot:

1. Legenerál egy ideiglenes `requirements.yml` fájlt a Git hitelesítési adatokkal.
2. Letölti a szükséges role-okat a lokális `roles/` könyvtárba.
3. Dinamikusan betölti a role-okat `include_role` segítségével.
4. A futás végén eltávolítja a generált requirements fájlt, akkor is, ha a playbook közben hibára fut.

```yaml
---
- name: Dinamikus Git-alapu role kezeles Galaxy szerver nelkul
  hosts: webservers
  become: true

  vars:
    # Javasolt: ezeket Ansible Vault-ban erdemes tarolni
    git_user: "{{ vault_git_user }}"
    git_token: "{{ vault_git_token }}"

    generated_requirements: "{{ playbook_dir }}/.requirements.generated.yml"
    local_roles_path: "{{ playbook_dir }}/roles"

  tasks:
    - name: Git-alapu role-ok telepitese es futtatasa
      block:
        - name: Ideiglenes requirements fajl generalasa Git hitelesitessel
          ansible.builtin.template:
            src: requirements.yml.j2
            dest: "{{ generated_requirements }}"
            mode: "0600"
          delegate_to: localhost
          run_once: true
          no_log: true

        - name: Role-ok letoltese a lokalis projekt konyvtarba
          ansible.builtin.command:
            cmd: >
              ansible-galaxy role install
              -r {{ generated_requirements }}
              -p {{ local_roles_path }}
              --force
          delegate_to: localhost
          run_once: true
          no_log: true

        - name: Nginx role dinamikus betoltese
          ansible.builtin.include_role:
            name: company.nginx

        - name: PostgreSQL role dinamikus betoltese
          ansible.builtin.include_role:
            name: company.postgresql

      always:
        - name: Ideiglenes requirements fajl torlese
          ansible.builtin.file:
            path: "{{ generated_requirements }}"
            state: absent
          delegate_to: localhost
          run_once: true
          no_log: true
```

---

## 🛠️ Miért így működik?

### Miért `include_role`?

Az `import_role` statikus működésű. Az Ansible már parse/compile időben feldolgozza, még azelőtt, hogy bármilyen task lefutna.

Ez azt jelenti, hogy ha a role még nincs jelen a lemezen a playbook indításakor, a playbook azonnal `Role not found` hibával leáll.

Az `include_role` ezzel szemben dinamikus. Futási időben értékelődik ki, akkor, amikor az adott task sorra kerül.

Ebben a mintában ez kulcsfontosságú, mert először le kell tölteni a role-okat, és csak utána lehet őket betölteni.

---

### Miért `ansible-galaxy role install`?

Dedikált Galaxy szerver nélkül is használható az `ansible-galaxy` CLI arra, hogy role-okat közvetlenül Git repository-kból telepítsen.

Itt tehát nem Galaxy szervert használunk, hanem az `ansible-galaxy` parancsot lokális dependency installer eszközként.

Ez egyszerűbbé teszi a role-ok kezelését, miközben nem kell külön Galaxy infrastruktúrát üzemeltetni.

---

### Miért kell a `--force`?

Ha a role könyvtár már létezik a lokális `roles/` mappában, az `ansible-galaxy` kihagyhatja az újratelepítést.

A `--force` biztosítja, hogy mindig a `requirements.yml` fájlban megadott verzió kerüljön alkalmazásra.

Ez akkor hasznos, ha egy role verzióját például `v3.1.0`-ról `v3.2.0`-ra módosítod, vagy commit SHA-ra váltasz.

Ennek az ára az, hogy a telepítési lépés gyakrabban tűnhet változottnak, mert a role könyvtár felülíródik.

---

### Miért kell verziót rögzíteni?

A verzió rögzítése reprodukálhatóbbá teszi a futást.

A playbook nem mindig az aktuális legfrissebb role állapotot tölti le, hanem egy konkrét Git taget vagy commit SHA-t.

Erősebb garanciához commit SHA vagy védett/signed Git tag használata javasolt. Egy sima Git tag elvileg mozgatható, ha a Git szerver oldalon nincs védve.

---

## Biztonsági szempontok

A Git hozzáférési adatokat nem szabad közvetlenül a repository-ban tárolni.

Javasolt gyakorlatok:

```text
- A Git felhasználót és tokent Ansible Vault-ban tárold.
- A hitelesítési adatokat tartalmazó requirements fájl csak futás közben jöjjön létre.
- A generált fájl jogosultsága legyen 0600.
- A titkokat kezelő taskokon legyen no_log: true.
- A generált fájlt always blokkban töröld.
- A generált fájlokat vedd fel a .gitignore-ba.
```

Példa `.gitignore`:

```gitignore
roles/
.requirements.generated.yml
requirements.yml
```

---

## Összefoglalás

Ez a megközelítés az Ansible role dependency-ket explicit, verziózott és Git-alapú módon kezeli.

Nem igényel dedikált Galaxy szervert, mégis lehetővé teszi, hogy a role-ok privát Git repository-kból, dinamikusan kerüljenek letöltésre.

A fő elv:

```text
Ne rejtsd el a role dependency-ket a role-ok belsejében.
Kezeld őket explicit módon projekt vagy playbook szinten.
```

Így a role-ok használata átláthatóbb, könnyebben auditálható, egyszerűbben frissíthető és reprodukálhatóbb lesz több környezet között.
