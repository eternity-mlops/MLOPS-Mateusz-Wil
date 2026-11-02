# Pracownia 3: Konteneryzacja lokalnego modelu LLM za pomocą Docker

## 1. Przygotowanie środowiska i zwolnienie zasobów

Przed wdrożeniem modelu w kontenerze zatrzymano i wyłączono usługę systemową `llamafile.service` z poprzedniego zadania, aby zwolnić port sieciowy `8080` oraz pamięć RAM:

```bash
sudo systemctl stop llamafile.service
sudo systemctl disable llamafile.service
```

Weryfikacja zwolnienia portu `8080` (brak wyniku potwierdza zamknięcie gniazda):

```bash
sudo ss -tlnp | grep 8080
```

## 2. Instalacja silnika Docker i konfiguracja uprawnień

Zainstalowano pakiety silnika Docker z oficjalnych repozytoriów systemu Ubuntu 24.04:

```bash
sudo apt update && sudo apt install -y docker.io
```

Dodano bieżącego użytkownika do grupy `docker`, aby umożliwić zarządzanie kontenerami bez konieczności używania uprawnień `sudo`:

```bash
sudo usermod -aG docker $USER
su - $USER
```

Weryfikacja poprawności instalacji i działania demona Docker:

```bash
docker --version
docker ps
```

## 3. Przygotowanie pliku Dockerfile i budowa obrazu

Utworzono katalog projektu `~/llama-docker` oraz plik `Dockerfile` definiujący środowisko uruchomieniowe dla modelu w oparciu o bazowy obraz `ubuntu:24.04`:

```bash
mkdir -p ~/llama-docker
cd ~/llama-docker
touch Dockerfile
nano Dockerfile
```

Zawartość pliku `Dockerfile`:

```dockerfile
FROM ubuntu:24.04

WORKDIR /app
ENV HOME=/app
ENV TMPDIR=/app

EXPOSE 8080

ENTRYPOINT ["/bin/sh", "/app/model.llamafile", "--server", "--nobrowser", "--no-mmap", "-t", "1", "-c", "2048", "--host", "0.0.0.0", "--port", "8080"]
```

**Opis dyrektyw użytych w `Dockerfile`:**
* `FROM ubuntu:24.04`: Wykorzystanie oficjalnego obrazu bazowego systemu Ubuntu 24.04.
* `WORKDIR /app` oraz `ENV HOME=/app`, `TMPDIR=/app`: Ustawienie katalogu roboczego i zmiennych środowiskowych wymaganych przez silnik `llamafile` do rozpakowania plików tymczasowych `.ape`.
* `EXPOSE 8080`: Deklaracja portu sieciowego, na którym serwer nasłuchuje wewnątrz kontenera.
* `ENTRYPOINT [...]`: Polecenie startowe kontenera uruchamiające serwer modelu z flagami zapewniającymi stabilność w środowisku wirtualnym (`--no-mmap`, `-t 1`, `-c 2048`) oraz nasłuchiwaniem na interfejsach wewnętrznych kontenera (`--host 0.0.0.0`).

Zbudowanie obrazu kontenera pod nazwą `llamafile-llm:v1`:

```bash
docker build -t llamafile-llm:v1 .
```

Weryfikacja utworzonego obrazu na liście lokalnych obrazów Docker:

```bash
docker images
```

Wynik polecenia `docker images` potwierdzający utworzenie obrazu[cite: 5]:

<img width="618" height="128" alt="image" src="https://github.com/user-attachments/assets/1ad5fe5a-2e63-47a1-8acd-96143445d505" />


## 4. Uruchomienie kontenera z modelem (Bind Mount)

Uruchomiono kontener `llama-container` w trybie tła (`-d`), montując plik modelu z systemu gospodarza jako wolumen typu *bind mount*:

```bash
docker run -d --name llama-container --restart unless-stopped -p 192.168.56.101:8080:8080 -v /opt/llamafile/model.llamafile:/app/model.llamafile llamafile-llm:v1
```

**Analiza użytych parametrów `docker run`:**
* `-d`: Uruchomienie kontenera w tle (*detached mode*).
* `--name llama-container`: Nadanie czytelnej nazwy instancji kontenera.
* `--restart unless-stopped`: Automatyczne wznawianie pracy kontenera po restarcie demona Docker lub maszyny wirtualnej.
* `-p 192.168.56.101:8080:8080`: Bindowanie portu `8080` kontenera wyłącznie do interfejsu sieci *Host-Only* (`192.168.56.101`) maszyny wirtualnej.
* `-v /opt/llamafile/model.llamafile:/app/model.llamafile`: Podpięcie pliku modelu z dysku maszyny wirtualnej bezpośrednio do przestrzeni plikowej kontenera (eliminuje konieczność duplikowania pliku o rozmiarze 670 MB wewnątrz obrazu).

Weryfikacja działającego kontenera oraz logów startowych serwera:

```bash
docker ps
docker logs llama-container | tail -n 15
```

## 5. Weryfikacja komunikacji z kontenerem przez REST API

Z poziomu maszyny gospodarza (Windows CMD) wykonano testowe zapytanie HTTP POST do skonteneryzowanej usługi na endpoint `/v1/chat/completions`:

```cmd
C:\Users\mateu>curl -X POST [http://192.168.56.101:8080/v1/chat/completions](http://192.168.56.101:8080/v1/chat/completions) -H "Content-Type: application/json" -d "{\"messages\":[{\"role\":\"user\",\"content\":\"Tell me a funny joke.\"}],\"max_tokens\":100}"
```

Odpowiedź zwrócona przez serwer modelu działający wewnątrz kontenera Docker (sformatowany JSON):

<img width="1150" height="154" alt="image" src="https://github.com/user-attachments/assets/e8423d6a-c625-4860-8474-2a48c2f09dea" />


```json
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
  "created": 1793627847,
  "id": "chatcmpl-dUhKy0oR6MwnUJzKqdSwtIL14IiFwrqz",
  "model": "unknown",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 100,
    "prompt_tokens": 30,
    "total_tokens": 130
  }
}
```
