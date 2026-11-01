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

1. Utworzenie dedykowanego katalogu w `/opt`:
   ```bash
   sudo mkdir -p /opt/llamafile
   cd /opt/llamafile
