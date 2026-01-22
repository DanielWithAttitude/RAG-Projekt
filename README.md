#  Projekt: Weather RAG na Klastrze k3s (Hyper-V)

Dokumentacja wdrożenia lokalnego klastra Kubernetes (k3s) na maszynach wirtualnych Hyper-V, służącego do przetwarzania danych pogodowych i generowania porad AI przy użyciu modelu LLM (Ollama).

## 1. Infrastruktura (Vagrant & Hyper-V)
```powershell
# przygotowanie hyper-v jeśli nie jest aktywne, może być wymagany restart
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
#sprawdzenie statusu hyper-v
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V

https://developer.hashicorp.com/vagrant/downloads - Instalacja vagranta
# Wykonujemy poniższą komendę z uprawnieniami admina
winrm quickconfig -q
```
Stawiamy dwie maszyny wirtualne na Hyper-V. Używamy Debian 12 z racji jego lekkości. Przy stawianiu maszyny używamy wirtualnego przełącznika "Default Switch"

### Uruchomienie maszyn

Bash

```yaml
# Uruchamiamy terminal jako administrator 
# Kopiujemy repozytorium na maszynę lokalną
git clone https://github.com/DanielWithAttitude/RAG-Projekt.git
# Przechodzimy do katalogu rag-projekt i wykonujemy poniższą komendę 
vagrant up --provider=hyperv
# W trakcie zostaniemy zapytani o przełącznik którego chcemy użyć. wybieramy liczbę odpowiadającą przełącznikowi
"Default Switch"

# Jeśli chcemy usunąć maszyny na którymś etapie testu należy wykonać z tego samego folderu polecenie:
vagrant destroy
```

---

## 2. Instalacja klastra k3s

Logujemy się przez SSH kluczem podanym w konfiguracji vagrant na maszyny i instalujemy binarki k3s.

### Na maszynie Master (`k3s-master`)

Instalujemy serwer, wyłączamy Traefik i ustawiamy uprawnienia do configu.
```bash
#SSH do mastera
vagrant ssh k3s-master
# Lub poprzez sesję ssh użytkownika vagrant, hasło vagrant
```


```bash
# SSH na mastera
curl -sfL https://get.k3s.io | sh -s - server --disable traefik --write-kubeconfig-mode 644

# Pobieramy token potrzebny dla workera
sudo cat /var/lib/rancher/k3s/server/node-token
# Pobieramy IP mastera
ip addr show eth0 | grep inet

```

### Na maszynie Worker (`k3s-worker`)

Podłączamy węzeł do klastra.

Bash

```bash
# Podmień <MASTER_IP> i <TOKEN> na wartości z poprzedniego kroku
# Wklejamy klucz w miejsce <TOKEN> i adres ip w miejsce <MASTER_IP>
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

### Weryfikacja

Wracamy na Mastera i sprawdzamy status:

Bash

```bash
kubectl get nodes
```

*Oczekiwany wynik: Oba węzły mają status `Ready`.*

---

## 3. Kafka (Strimzi Operator)

Do obsługi Kafki używamy operatora Strimzi. Tworzymy klaster w trybie **KRaft** (bez Zookeepera).

### Instalacja Operatora

Bash

```bash
# Wszystkie komendy kubernetesowe wykonujemy na k3s-master
# Tworzymy namespace dla kafki
kubectl create namespace kafka
# Wdrożenie dystrybucji kafka strimzi
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl get pods -A
```

### Definicja klastra i topica (`kafka-cluster.yaml`)

Plik zawiera definicję puli węzłów, konfigurację klastra Kafka oraz Topic `weather-data`.

**Uruchomienie:**
Bash
```bash
# Przekopiowanie pliku kafka-cluster.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp kafka-cluster.yaml vagrant@<MASTER_IP>:
# Następnie wykonujemy poniższą komendę 
kubectl apply -f kafka-cluster.yaml
# Czekamy aż pod kafki wstanie
kubectl get pods -n kafka
kubectl get pods -A
```

---

## 4. Backend AI (Ollama)

Wdrażamy lokalny model językowy, który będzie naszym "mózgiem".

### Plik: `ollama-deployment.yaml`
**Uruchomienie:**
Bash

```bash
# Przekopiowanie ollama-deployment.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp ollama-deployment.yaml vagrant@<MASTER_IP>:
# Następnie wykonujemy poniższą komendę 
kubectl apply -f ollama-deployment.yaml
kubectl get pods -A
```

---

## 5. Logika Aplikacji (Producer & Consumer)

Kod aplikacji Python trzymamy w `ConfigMap`. Dzięki temu możemy go edytować bez przebudowywania obrazów Docker.

- **Producer:** Pobiera pogodę z darmowego api Open-Meteo i wrzuca na odpowiedni topic do Kafki.
- **Consumer:** Czyta z Kafki, pyta Ollamę o odpowiedź w oparciu o kontekst.

### Plik: `app-code.yaml`
**Uruchomienie:**
```bash
# Przekopiowanie app-code.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp app-code.yaml vagrant@<MASTER_IP>: 
# Następnie wykonujemy poniższą komendę 
kubectl apply -f app-code.yaml
kubectl get pods -A
```

---

## 6. Wdrożenie (Deployment)

Tutaj tworzymy Pody, które faktycznie uruchamiają powyższy kod. Używamy obrazu `python:3.9-slim` i doinstalowujemy zależności (`kafka-python`, `requests`) w locie przy starcie kontenera (tzw. command override).

### Plik: `rag-apps.yaml`
**Uruchomienie:**

```bash
# Przekopiowanie rag-apps.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp rag-apps.yaml vagrant@<MASTER_IP>: 
kubectl apply -f rag-apps.yaml
```

**Sprawdzenie czy działa:**

```bash
kubectl get pods -A
```

---

## 7. Pod kliencki
Tworzymy Pod który będzie korzystał z kontekstu i generował odpowiedź w oparciu o zapytanie do modelu ollama

### Plik: `client-config.yaml`
**Uruchomienie:**

```bash
# Przekopiowanie client-config.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp client-config.yaml vagrant@<MASTER_IP>:
kubectl apply -f client-config.yaml
```
**Sprawdzenie czy działa:**

```bash
kubectl get pods -A
```

### Plik: `client-config.yaml`
**Uruchomienie:**

```bash
# Przekopiowanie rag-client-deployment.yaml z katalogu projektu na maszynę k3s-master
# W miejsce <MASTER_IP> podajemy adres ip maszyny k3s-master
scp rag-client-deployment.yaml vagrant@<MASTER_IP>:
kubectl apply -f rag-client-deployment.yaml
```
**Sprawdzenie czy działa:**

```bash
kubectl get pods -A
```

## 8. Test przy użyciu poda klienckiego
bash
```
# Uruchamiamy aplikację poniższą komendą, nazwa poda będzie inna za kazdym utworzeniem poda
# Najpierw wykonujemy kubectl get pods -A i kopiujemy nazwę rag-client-XXXXXXXXXX-XXXX
# W moim przypadku pod nazywa się rag-client-788db78649-xsv7d
kubectl exec -it <NAZWA_PODA_RAG_CLIENT> -- python /app/rag_app.py
```


Przykład działania aplikacji:
<img width="1021" height="485" alt="image" src="https://github.com/user-attachments/assets/93481ae8-6dbe-49c5-9861-1fe34206e152" />
