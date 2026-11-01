# Dokumentacja: Wdrożenie modelu llamafile jako usługi systemd

Zadanie polega na uruchomieniu gotowego modelu uczenia maszynowego w postaci pliku llamafile jako usługi systemd na maszynie wirtualnej z systemem Linux, dostępnej wyłącznie przez sieć Host-Only.

---

## 1. Przygotowanie środowiska i pobranie modelu

Rozpoczęto od połączenia SSH z maszyną wirtualną (Ubuntu) oraz pobrania pliku wykonywalnego modelu do dedykowanego katalogu. Wybrano model TinyLlama 1.1B w kwantyzacji Q4_K_M.

* `ssh vboxuser@192.168.56.101`
  Logowanie przez SSH na porcie 22.
* `sudo mkdir -p /opt/llamafile`
  Utworzenie katalogu dla usługi.
* `sudo curl -L -o /opt/llamafile/model.llamafile https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile`
  Pobranie modelu językowego z repozytorium HuggingFace.
  * **Flaga `-L` (Location):** Podąża za przekierowaniami HTTP.
  * **Flaga `-o` (Output):** Zapisuje pobrany strumień danych pod wskazaną nazwą i ścieżką.

---

## 2. Konto oraz uprawnienia

Usługa działa z uprawnieniami dedykowanego użytkownika bez praw roota:

* `sudo useradd --system --no-create-home llamafile`
  * **Flaga `--system`:** Tworzy konto systemowe bez możliwości logowania na powłokę.
  * **Flaga `--no-create-home`:** Zapobiega tworzeniu zbędnego katalogu domowego w `/home/`.
* `sudo chmod +x /opt/llamafile/model.llamafile`
  Nadanie atrybutu wykonywalności dla pliku binarnego.
* `sudo chown -R llamafile:llamafile /opt/llamafile`
  Zmiana właściciela katalogu i pliku na użytkownika `llamafile`.

---

## 3. Konfiguracja jednostki systemd

Plik jednostki `/etc/systemd/system/llamafile.service`:

```ini
[Unit]
Description=Llamafile Service
After=network-online.target
Wants=network-online.target

[Service]
User=llamafile
WorkingDirectory=/opt/llamafile
Environment="HOME=/opt/llamafile"
Environment="TMPDIR=/opt/llamafile"
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --nobrowser -t 1 -c 512 --host 192.168.56.101 --port 8080
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
