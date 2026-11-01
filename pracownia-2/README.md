# Pracownia 2 – Wdrożenie lokalnego modelu językowego llamafile jako usługi systemd

Dokumentacja opisuje procedurę wdrożenia gotowego modelu uczenia maszynowego w architekturze Linuksa z wykorzystaniem technologii `llamafile` oraz menedżera usług `systemd`. Całość została skonfigurowana zgodnie z zasadą minimalnych uprawnień (brak uprawnień root) oraz odizolowana sieciowo wyłącznie do interfejsu Host-Only.

---

## 1. Wymagania wstępne i adresacja środowiska

*   **System operacyjny:** Ubuntu 24.04 LTS (maszyna wirtualna VirtualBox).
*   **Architektura sieciowa:**
    *   `enp0s3` (NAT): dostęp do internetu do pobierania pakietów i wag.
    *   `enp0s8` (Host-Only): **192.168.56.101** – dedykowany interfejs do komunikacji z maszyną gospodarza (Windows).
    *   `enp0s9` (Internal): sieć laboratoryjna.
*   **Klient:** Połączenie administracyjne realizowane z maszyny gospodarza przez SSH:
    ```bash
    ssh vboxuser@192.168.56.101
    ```

---

## 2. Przygotowanie katalogu i pobranie modelu (curl)

Format `llamafile` łączy w jednym pliku binarnym silnik wykonawczy C++, wagi sieci neuronowej oraz wielowątkowy serwer HTTP z API kompatybilnym z OpenAI.

Jako model wybrano **TinyLlama-1.1B-Chat-v1.0** skwantyzowany do formatu `Q4_K_M` (4-bitowa kwantyzacja wag, rozmiar ~670 MB), co zapewnia niski narzut pamięciowy i brak ryzyka przeciążenia procesora.

Tworzymy dedykowany katalog w `/opt`:
```bash
sudo mkdir -p /opt/llamafile
cd /opt/llamafile
```
   
Pobieramy plik binarny z repozytorium HuggingFace:

```bash
sudo curl -L -o /opt/llamafile/model.llamafile https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile
```

Wyjaśnienie użytych parametrów:
* -L (--location): Wymusza podążanie za przekierowaniami HTTP/HTTPS (niezbędne w przypadku HuggingFace CDN)
* -o (--output): Określa docelową ścieżkę zapisu pliku na dysku.

## 3. Utworzenie użytkownika usługowego i uprawnienia
Zgodnie z zasadami bezpieczeństwa usługa nie może działać na koncie root. Wyciek pamięci lub podatność serwera HTTP nie może skutkować przejęciem kontroli nad jądrem systemu.

1. Utworzenie użytkownika systemowego:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin llamafile
Wyjaśnienie parametrów:
```

Wyjaśnienie parametrów:
* --system: Tworzy konto systemowe z identyfikatorem UID < 1000, niewidoczne na ekranie powitalnym środowiska graficznego.
* --no-create-home: Blokuje tworzenie katalogu domowego w /home/.
* --shell /usr/sbin/nologin: Uniemożliwia bezpośrednie zalogowanie się na to konto przez SSH lub terminal.

2. Nadanie praw wykonywalności i zmiana własności:

```bash
sudo chmod +x /opt/llamafile/model.llamafile
sudo chown -R llamafile:llamafile /opt/llamafile
chmod +x: Nadaje atrybut wykonywalności, umożliwiając uruchomienie pliku jako programu binarnego.
```

* chmod +x: Nadaje atrybut wykonywalności, umożliwiając uruchomienie pliku jako programu binarnego.
* chown -R: Rekursywnie przypisuje katalog i zawarty w nim model do nowego użytkownika i grupy llamafile

## 4. Konfiguracja jednostki systemd (/etc/systemd/system/llamafile.service)
Utworzono plik definicji usługi za pomocą edytora tekstu:

```bash
sudo nano /etc/systemd/system/llamafile.service
```
Zawartość pliku jednostki:

```bash
[Unit]
Description=Llamafile Local LLM Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=llamafile
Group=llamafile
WorkingDirectory=/opt/llamafile
Environment="HOME=/opt/llamafile"
Environment="TMPDIR=/opt/llamafile"
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --nobrowser -t 1 -c 512 --host 192.168.56.101 --port 8080
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Analiza parametrów uruchomieniowych:
* After=network-online.target / Wants=...: Usługa wystartuje dopiero po pełnym podniesieniu stosu sieciowego.
* User=llamafile / Group=llamafile: Izolacja uprawnień procesu w systemie.
* WorkingDirectory, HOME, TMPDIR: Wskazanie lokalizacji roboczej, gdzie silnik może tworzyć tymczasowe bufory cache.
* --server --nobrowser: Tryb serwerowy bez próby wywoływania przeglądarki na maszynie bezgłowej.
* -t 1: Ograniczenie egzekucji do dokładnie 1 wątku CPU (zapobiega nasyceniu rdzeni vCPU i zawieszaniu maszyny wirtualnej).
* -c 512: Ograniczenie rozmiaru bufora kontekstu do 512 tokenów w celu drastycznej redukcji alokacji pamięci RAM.
* --host 192.168.56.101 --port 8080: Sztywne bindowanie gniazda sieciowego wyłącznie do interfejsu Host-Only.
* Restart=on-failure: Automatyczne przywracanie procesu w przypadku awarii lub nieoczekiwanego zakończenia (np. sygnał SIGKILL).
* RestartSec=5s: Odczekanie 5 sekund przed ponowną próbą podniesienia procesu.

## 5. Przeładowanie konfiguracji, autostart i uruchomienie
Wczytanie nowego pliku jednostki do demona systemd:

```bash
sudo systemctl daemon-reload
```
Włączenie autostartu przy rozruchu maszyny oraz natychmiastowe uruchomienie:

```bash
sudo systemctl enable --now llamafile.service
```
Weryfikacja działania w czasie rzeczywistym (opcjonalny podgląd dziennika zdarzeń):

```bash
journalctl -u llamafile.service -f
```
