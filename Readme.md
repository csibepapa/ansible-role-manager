# Git-Alapú Dinamikus Szerepkör Kezelés Ansible Galaxy Szerver Nélkül

Ez a produkcióra kész (production-ready) megoldás lehetővé teszi, hogy az Ansible szerepköröket (roles) közvetlenül a saját Git szerveredről töltsd le, miközben a Git hozzáférési tokeneket biztonságosan, dinamikusan injektálod be a futás pillanatában. 

A fix verziók (pinning) garantálják az atomstabil működést, az `include_role` pedig biztosítja, hogy a playbook ne hasaljon el fordítási fázisban.

## 📁 Ajánlott könyvtárstruktúra

A projekt elszigeteltsége (flat design) érdekében érdemes az alábbi struktúrát követni. A `roles/` mappa automatikusan létrejön és feltöltődik a futás során.

```text
my-ansible-project/
├── requirements.yml.j2    # Jinja2 sablon a Git token behelyettesítéséhez
├── playbook.yml           # A fő playbook (ezt futtatod)
└── roles/                 # Ide fognak letöltődni a szerepkörök (automatikus)
```

---

## 📄 Forráskódok

### 1. `requirements.yml.j2` (Sablon fájl)
Ez a sablon végzi el a Git hitelesítési adatok behelyettesítését. A verziók fix Git Tag-ekre vannak rögzítve az idempotencia miatt.

```yaml
---
roles:
  - name: geerlingguy.nginx
    src: "git+https://{{ git_user }}:{{ git_token }}@://github.com"
    version: "v3.1.0"  # Fix Git Tag

  - name: geerlingguy.postgresql
    src: "git+https://{{ git_user }}:{{ git_token }}@://github.com"
    version: "v3.4.0"  # Fix Git Tag
```

### 2. `playbook.yml` (Fő playbook)
Ez a fájl vezérli a teljes folyamatot. A `pre_tasks` szakasz legenerálja a fájlt a tokenekkel, letölti a szerepköröket a helyi `./roles/` mappába, a `tasks` szakaszban lévő `include_role` pedig dinamikusan végrehajtja őket. A folyamat végén egy `post_tasks` biztonsági okokból letörli a legenerált fájlt, hogy a token ne maradjon a lemezen.

```yaml
---
- name: Dinamikus Git alapu role kezeles Galaxy nelkul
  hosts: webservers
  become: yes

  vars:
    # Biztonsági jótanács: Ezeket a változókat érdemes Ansible Vault-ban titkosítani!
    git_user: "my_git_username"
    git_token: "ghp_DeMoToKeN1234567890"

  pre_tasks:
    - name: Requirements fajl legeneralasa a Git adatokkal
      ansible.builtin.template:
        src: requirements.yml.j2
        dest: requirements.yml
      delegate_to: localhost
      run_once: true

    - name: Szerepkorok letoltese a lokalis projekt mappaba
      ansible.builtin.command:
        cmd: ansible-galaxy role install -r requirements.yml -p ./roles/ --force
      delegate_to: localhost
      run_once: true
      register: galaxy_output
      changed_when: "'is already installed' not in galaxy_output.stdout"

  tasks:
    - name: Nginx szerepkor dinamikus betoltese (Futasi idoben)
      ansible.builtin.include_role:
        name: geerlingguy.nginx

    - name: Postgresql szerepkor dinamikus betoltese (Futasi idoben)
      ansible.builtin.include_role:
        name: geerlingguy.postgresql

  post_tasks:
    - name: Biztonsagi takaritas - requirements.yml torlese a localhostrol
      ansible.builtin.file:
        path: requirements.yml
        state: absent
      delegate_to: localhost
      run_once: true
```

---

## 🛠️ Miért pont így működik? (Technikai háttér)

* **Miért KÖTELEZŐ az `include_role`?**Az Ansible alapértelmezetten az `import_role`-t fordítási időben (compile-time) olvassa be, még a feladatok futása előtt. Mivel a playbook indításakor a `requirements.yml` még nem létezik és a szerepkörök nincsenek letöltve, az `import_role` azonnali `Role not found` hibával leállítaná a futást. Az `include_role` ezzel szemben futási időben (runtime) értékelődik ki. Megvárja, amíg a `pre_tasks` letölti a Git repókat, és csak akkor keresi őket a lemezen, amikor a sor rákerül.
* **Miért kell a `--force` flag?** Mivel a verziók fixálva vannak, ha a jövőben módosítod a verziót a sablonban (pl. frissítesz `v3.5.0`-re), az `ansible-galaxy` parancs a `--force` nélkül látná, hogy a mappa már létezik, és kihagyná a letöltést. A `--force` kényszeríti a tiszta felülírást.
* **Biztonság és hordozhatóság:** A tokenek csak átmenetileg, a futás idejére kerülnek ki a lemezre a `localhost`-on. A `post_tasks` takarító lépése miatt a szenzitív adatok nem maradnak ott a fájlrendszeren, és a Git verziókövetőbe (pl. GitHub/GitLab) sem fognak bekerülni.

