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
```
Wyjaśnienie parametrów:

Wyjaśnienie parametrów:
* --system: Tworzy konto systemowe z identyfikatorem UID < 1000, niewidoczne na ekranie powitalnym środowiska graficznego.
* --no-create-home: Blokuje tworzenie katalogu domowego w /home/.
* --shell /usr/sbin/nologin: Uniemożliwia bezpośrednie zalogowanie się na to konto przez SSH lub terminal.

2. Nadanie praw wykonywalności i zmiana własności:

```bash
sudo chmod +x /opt/llamafile/model.llamafile
sudo chown -R llamafile:llamafile /opt/llamafile
```
* chmod +x: Nadaje atrybut wykonywalności, umożliwiając uruchomienie pliku jako programu binarnego.
* chown -R: Rekursywnie przypisuje katalog i zawarty w nim model do nowego użytkownika i grupy llamafile

## 4. Konfiguracja jednostki systemd (/etc/systemd/system/llamafile.service)
Utworzono plik definicji usługi za pomocą edytora tekstu:

```bash
sudo nano /etc/systemd/system/llamafile.service
```
Zawartość pliku jednostki:

```ini
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
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --nobrowser --no-mmap -t 1 -c 2048 --host 192.168.56.101 --port 8080
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**Analiza parametrów uruchomieniowych:**
* `After=network-online.target` / `Wants=...`: Usługa wystartuje dopiero po pełnym podniesieniu stosu sieciowego.
* `User=llamafile` / `Group=llamafile`: Izolacja uprawnień procesu w systemie (zasada *least privilege*).
* `WorkingDirectory`, `HOME`, `TMPDIR`: Wskazanie lokalizacji roboczej, gdzie silnik może tworzyć tymczasowe pliki wykonywalne `.ape` oraz bufory cache.
* `--server --nobrowser`: Uruchomienie w trybie serwera HTTP bez próby wywoływania przeglądarki graficznej na maszynie typu *headless*.
* `--no-mmap`: Wymuszenie bezpośredniego załadowania wag modelu do pamięci RAM zamiast mapowania pliku z wirtualnego dysku (eliminuje opóźnienia I/O w VirtualBox).
* `-t 1`: Ograniczenie egzekucji do 1 wątku CPU (zapobiega zakleszczaniu wątków warstwy APE pod hiperwizorem i zawieszaniu maszyny wirtualnej).
* `-c 2048`: Ustawienie rozmiaru bufora kontekstu na 2048 tokenów, co pozwala na swobodną obsługę dłuższych zapytań i historii czatu.
* `--host 192.168.56.101 --port 8080`: Sztywne bindowanie gniazda sieciowego wyłącznie do interfejsu sieci *Host-Only*.
* `Restart=on-failure`: Automatyczne przywracanie procesu w przypadku awarii lub nieoczekiwanego zakończenia (np. sygnał `SIGKILL`).
* `RestartSec=5s`: Odczekanie 5 sekund przed ponowną próbą podniesienia procesu.

## 5. Przeładowanie konfiguracji, autostart i uruchomienie
1. Wczytanie nowego pliku jednostki do demona systemd:

```bash
sudo systemctl daemon-reload
```
2. Włączenie autostartu przy rozruchu maszyny oraz natychmiastowe uruchomienie:

```bash
sudo systemctl enable --now llamafile.service
```
3. Weryfikacja działania w czasie rzeczywistym (opcjonalny podgląd dziennika zdarzeń):

```bash
journalctl -u llamafile.service -f
```

## 6. Weryfikacja wymagań technicznych
**A. Weryfikacja nasłuchu na porcie i izolacji sieciowej (ss -tlnp)**
Sprawdzenie stanu otwartych portów TCP:

<img width="864" height="48" alt="image" src="https://github.com/user-attachments/assets/37c4296a-904e-416b-9447-9c8e296d02a2" />

Interpretacja: Usługa nasłuchuje pod adresem 192.168.56.101:8080. Adresy 127.0.0.1 (loopback), 10.0.2.15 (NAT) oraz interfejs wewnętrzny nie publikują tego portu, co potwierdza spełnienie wymogu izolacji.

**B. Test odporności na awarie (automatyczny restart po kill -9)**
Test mechanizmu self-healing realizowanego przez systemd:

Wysłanie sygnału natychmiastowego zabicia procesu (SIGKILL) do aktywnego PID:

<img width="1144" height="268" alt="image" src="https://github.com/user-attachments/assets/091786ee-b0b1-4f0f-bd14-7cb1e6ad0fa0" />

Interpretacja: Systemd odnotował nieprawidłowe zatrzymanie procesu i samoczynnie uruchomił usługę pod nowym identyfikatorem PID: 5584.

**C. Test komunikacji REST API z maszyny gospodarza (curl)**
Wykonanie zapytania HTTP POST z poziomu terminala systemu gospodarza (Windows PowerShell) w celu weryfikacji generowania tokenów:

```bash
curl -X POST http://192.168.56.101:8080/v1/chat/completions -H "Content-Type: application/json" -d "{\"messages\":[{\"role\":\"user\",\"content\":\"Tell me a funny joke.\"}],\"max_tokens\":100}"
```
Odpowiedź zwrócona przez model (JSON):

```JSON
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "A man walks into a bar and orders a drink. The bartender looks at him skeptically and asks, \"What's in that drink?\" The man replies, \"Just a little bit of luck.\" The bartender smiles and says, \"Well, that's a good start. But you'll need more.\" The man looks at the bartender, confused, and says, \"What do you mean, more? I'm just ordering a",
        "role": "assistant"
      }
    }
  ],
  "created": 1793565899,
  "id": "chatcmpl-1o3GfVKFJAi9LaB5gbXR9w6vueU97jV0",
  "model": "unknown",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 100,
    "prompt_tokens": 30,
    "total_tokens": 130
  }
}
```
