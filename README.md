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

### Plik: `Vagrantfile`

> 

```yaml
VAGRANTFILE_API_VERSION = "2"

NODES = {
  "k3s-master" => { cpus: 2, memory: 2048 },
  "k3s-worker" => { cpus: 2, memory: 2048 }
}

if !File.exist?("./vagrant_rsa")
  system("ssh-keygen -t rsa -b 2048 -f ./vagrant_rsa -q -N ''")
end
SSH_PUB_KEY = File.read("./vagrant_rsa.pub").strip

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "generic/debian12"
  config.ssh.insert_key = false
  config.ssh.private_key_path = ["./vagrant_rsa", "~/.vagrant.d/insecure_private_key"]
  config.vm.synced_folder ".", "/vagrant", disabled: true

  NODES.each do |name, opts|
    config.vm.define name do |node|
      node.vm.hostname = name

      node.vm.provider "hyperv" do |hv|
        hv.vmname       = name
        hv.cpus         = opts[:cpus]
        hv.memory       = opts[:memory]
        hv.linked_clone = true
        hv.enable_virtualization_extensions = true
      end

      node.vm.provision "shell", inline: <<-SHELL
        set -euxo pipefail

        # Konfiguracja SSH (klucz publiczny)
        install -d -m 700 /root/.ssh
        touch /root/.ssh/authorized_keys
        chmod 600 /root/.ssh/authorized_keys
        grep -qxF '#{SSH_PUB_KEY}' /root/.ssh/authorized_keys || echo '#{SSH_PUB_KEY}' >> /root/.ssh/authorized_keys
        
        if grep -qE '^#?PermitRootLogin' /etc/ssh/sshd_config; then
          sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
        else
          echo 'PermitRootLogin prohibit-password' >> /etc/ssh/sshd_config
        fi
        systemctl restart ssh
      SHELL
    end
  end
end
```

### Uruchomienie maszyn

Bash

```yaml
vagrant up --provider=hyperv
```

---

## 2. Instalacja klastra k3s

Logujemy się przez SSH kluczem podanym w konfiguracji vagrant na maszyny i instalujemy binarki k3s.

### Na maszynie Master (`k3s-master`)

Instalujemy serwer, wyłączamy Traefik i ustawiamy uprawnienia do configu.
```bash
#SSH do mastera
vagrant ssh k3s-master
```


```bash
# SSH na mastera
curl -sfL https://get.k3s.io | sh -s - server --disable traefik --write-kubeconfig-mode 644

# Pobieramy token potrzebny dla workera
cat /var/lib/rancher/k3s/server/node-token
# Pobieramy IP mastera
ip addr show eth0 | grep inet

```

### Na maszynie Worker (`k3s-worker`)

Podłączamy węzeł do klastra.

Bash

```bash
# Podmień <MASTER_IP> i <TOKEN> na wartości z poprzedniego kroku
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
kubectl create namespace kafka
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl get pods -A
```

### Definicja klastra i topica (`kafka-cluster.yaml`)

Plik zawiera definicję puli węzłów, konfigurację klastra Kafka oraz Topic `weather-data`.

YAML

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: dual-role
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 1
  roles:
    - controller
    - broker
  storage:
    type: ephemeral
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
  annotations:
    strimzi.io/node-pools: "enabled"
    strimzi.io/kraft: "enabled"
spec:
  kafka:
    version: 4.0.0
    metadataVersion: 4.0-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: weather-data
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 1
  replicas: 1
```

**Uruchomienie:**

Bash

```bash
kubectl apply -f kafka-cluster.yaml
# Czekamy aż pod kafki wstanie
kubectl get pods -n kafka
kubectl get pods -A
```

---

## 4. Backend AI (Ollama)

Wdrażamy lokalny model językowy, który będzie naszym "mózgiem".

### Plik: `ollama-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "sleep 5; ollama serve & sleep 5; ollama pull tinyllama"]
---
apiVersion: v1
kind: Service
metadata:
  name: ollama-service
  namespace: default
spec:
  selector:
    app: ollama
  ports:
    - protocol: TCP
      port: 80
      targetPort: 11434
```

**Uruchomienie:**

Bash

```bash
kubectl apply -f ollama-deployment.yaml
kubectl get pods -A
```

---

## 5. Logika Aplikacji (Producer & Consumer)

Kod aplikacji Python trzymamy w `ConfigMap`. Dzięki temu możemy go edytować bez przebudowywania obrazów Docker.

- **Producer:** Pobiera pogodę z darmowego api Open-Meteo i wrzuca na odpowiedni topic do Kafki.
- **Consumer:** Czyta z Kafki, pyta Ollamę o odpowiedź w oparciu o kontekst.

### Plik: `app-code.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rag-scripts
  namespace: default
data:
  producer.py: |
    import time
    import json
    import requests
    from kafka import KafkaProducer
    from kafka.errors import NoBrokersAvailable

    def create_producer():
        while True:
            try:
                print("Proba polaczenia z Kafka...")
                producer = KafkaProducer(
                    bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc:9092',
                    value_serializer=lambda v: json.dumps(v).encode('utf-8')
                )
                print("Polaczono z Kafka!")
                return producer
            except NoBrokersAvailable:
                print("Kafka niedostepna. Ponawiam za 5 sekund...")
                time.sleep(5)
            except Exception as e:
                print(f"Inny blad Kafki: {e}. Ponawiam...")
                time.sleep(5)

    producer = create_producer()

    LOCATIONS = [
        {"city": "Bialystok", "lat": 53.1325, "lon": 23.1688},
        {"city": "Bydgoszcz", "lat": 53.1235, "lon": 18.0084},
        {"city": "Gdansk", "lat": 54.3520, "lon": 18.6466},
        {"city": "Gorzow Wlkp", "lat": 52.7368, "lon": 15.2288},
        {"city": "Katowice", "lat": 50.2584, "lon": 19.0275},
        {"city": "Kielce", "lat": 50.8703, "lon": 20.6275},
        {"city": "Krakow", "lat": 50.0647, "lon": 19.9450},
        {"city": "Lublin", "lat": 51.2465, "lon": 22.5684},
        {"city": "Lodz", "lat": 51.7592, "lon": 19.4560},
        {"city": "Olsztyn", "lat": 53.7784, "lon": 20.4801},
        {"city": "Opole", "lat": 50.6751, "lon": 17.9213},
        {"city": "Poznan", "lat": 52.4064, "lon": 16.9252},
        {"city": "Rzeszow", "lat": 50.0413, "lon": 21.9990},
        {"city": "Szczecin", "lat": 53.4289, "lon": 14.5530},
        {"city": "Torun", "lat": 53.0138, "lon": 18.5984},
        {"city": "Warszawa", "lat": 52.2297, "lon": 21.0122},
        {"city": "Wroclaw", "lat": 51.1079, "lon": 17.0385},
        {"city": "Zielona Gora", "lat": 51.9356, "lon": 15.5062}
    ]

    def get_weather_description(code):
        if code == 0: return "Bezchmurnie"
        if 1 <= code <= 3: return "Czesciowe zachmurzenie"
        if 45 <= code <= 48: return "Mgla"
        if 51 <= code <= 55: return "Mzawka"
        if 61 <= code <= 65: return "Deszcz"
        if 71 <= code <= 75: return "Snieg"
        if 95 <= code <= 99: return "Burza"
        return "Nieznana pogoda"

    def fetch_weather(location):
        try:
            url = f"https://api.open-meteo.com/v1/forecast?latitude={location['lat']}&longitude={location['lon']}&current_weather=true"
            response = requests.get(url, timeout=5)
            data = response.json()

            if 'current_weather' in data:
                cw = data['current_weather']
                return {
                    "city": location['city'],
                    "temp": cw['temperature'],
                    "wind_speed": cw['windspeed'],
                    "condition": get_weather_description(cw['weathercode'])
                }
        except Exception as e:
            print(f"Blad pobierania pogody dla {location['city']}: {e}")
            return None

    print("Startuje pętle wysylania danych...")
    while True:
        for loc in LOCATIONS:
            weather_data = fetch_weather(loc)
            if weather_data:
                print(f"Wysylam dane: {weather_data}")
                producer.send('weather-data', weather_data)

        print("Czekam 60 sekund...")
        time.sleep(60)

  consumer_rag.py: |
    import json
    import time
    import requests
    from kafka import KafkaConsumer
    from kafka.errors import NoBrokersAvailable

    print("Startuje RAG Consumer...")

    def create_consumer():
        while True:
            try:
                print("Proba polaczenia z Kafka...")
                consumer = KafkaConsumer(
                    'weather-data',
                    bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc:9092',
                    auto_offset_reset='latest',
                    value_deserializer=lambda x: json.loads(x.decode('utf-8'))
                )
                print("Polaczono z Kafka!")
                return consumer
            except NoBrokersAvailable:
                print("Kafka niedostepna. Ponawiam za 5 sekund...")
                time.sleep(5)
            except Exception as e:
                print(f"Inny blad Kafki: {e}. Ponawiam...")
                time.sleep(5)

    consumer = create_consumer()
    OLLAMA_URL = "http://ollama-service.default.svc:80/api/generate"

    for message in consumer:
        w = message.value
        print(f"Otrzymano LIVE pogode: {w}")

        context = (f"Miasto: {w['city']}. "
                   f"Temperatura: {w['temp']} stopni Celsjusza. "
                   f"Warunki: {w['condition']}. "
                   f"Predkosc wiatru: {w['wind_speed']} km/h.")

        user_query = f"Jestem w miescie {w['city']}, jak powinienem sie ubrac i czy brac parasol?"
        full_prompt = f"Context: {context}\\nQuestion: {user_query}\\nAnswer as a helpful assistant based strictly on the context provided. Keep it short."

        payload = {
            "model": "tinyllama",
            "prompt": full_prompt,
            "stream": False
        }

        try:
            response = requests.post(OLLAMA_URL, json=payload, timeout=30)
            ai_text = response.json().get('response', '')
            print(f"--- AI Porada dla {w['city']} ---\\n{ai_text}\\n---------------------------")
        except Exception as e:
            print(f"Blad polaczenia z LLM: {e}")
```

**Uruchomienie:**

```bash
kubectl apply -f app-code.yaml
kubectl get pods -A
```

---

## 6. Wdrożenie (Deployment)

Tutaj tworzymy Pody, które faktycznie uruchamiają powyższy kod. Używamy obrazu `python:3.9-slim` i doinstalowujemy zależności (`kafka-python`, `requests`) w locie przy starcie kontenera (tzw. command override).

### Plik: `rag-apps.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-producer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: weather-producer
  template:
    metadata:
      labels:
        app: weather-producer
    spec:
      containers:
      - name: producer
        image: python:3.9-slim
        command: ["/bin/sh", "-c"]
        args:
          - pip install kafka-python requests && python /app/producer.py
        volumeMounts:
        - name: script-vol
          mountPath: /app
      volumes:
      - name: script-vol
        configMap:
          name: rag-scripts
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rag-consumer
  template:
    metadata:
      labels:
        app: rag-consumer
    spec:
      containers:
      - name: consumer
        image: python:3.9-slim
        command: ["/bin/sh", "-c"]
        args:
          - pip install kafka-python requests && python /app/consumer_rag.py
        volumeMounts:
        - name: script-vol
          mountPath: /app
      volumes:
      - name: script-vol
        configMap:
          name: rag-scripts
```

**Uruchomienie:**

```bash
kubectl apply -f rag-apps.yaml
```

**Sprawdzenie czy działa:**

```bash
kubectl get pods -A
```

---

## 7. Testowanie interaktywne

Aby nie tylko patrzeć w logi, uruchamiamy interaktywnego klienta, który pozwala wybrać miasto i zadać pytanie do AI na żywo.

1. Uruchom tymczasowy pod testowy:Bash
    
    ```bash
    kubectl run -it --rm interactive-test --image=python:3.9-slim --restart=Never -- /bin/bash
    ```
    
2. Wewnątrz poda zainstaluj biblioteki:
    
    ```bash
    pip install kafka-python requests
    ```
    
3. Utwórz plik ze skryptem testowym - wystarczy wkleic do poda klienta w którym aktualnie się znajdujemy

```bash
cat <<EOF > test_rag_full.py
import json
import requests
import sys
import uuid
from kafka import KafkaConsumer

CITIES = [
    "Bialystok", "Bydgoszcz", "Gdansk", "Gorzow Wlkp", "Katowice",
    "Kielce", "Krakow", "Lublin", "Lodz", "Olsztyn", 
    "Opole", "Poznan", "Rzeszow", "Szczecin", "Torun", 
    "Warszawa", "Wroclaw", "Zielona Gora"
]

def get_weather_for_city(target_city):
    print(f"\n>>> Szukam najnowszych danych dla: {target_city}...")
    print("(Czekaj, az producer wysle dane...)")

    # Tworzymy nowego consumera dla kazdego zapytania o miasto, 
    # zeby miec pewnosc ze bierzemy 'swieze' dane, a nie stare z bufora
    consumer = KafkaConsumer(
        'weather-data',
        bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc:9092',
        auto_offset_reset='latest',
        group_id=f"tester-{uuid.uuid4()}",
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )

    weather_data = None
    for message in consumer:
        data = message.value
        # print(f" [skanuje: {data['city']}]", end="\r") # Opcjonalny podglad
        if data['city'] == target_city:
            weather_data = data
            break 
    
    consumer.close()
    return weather_data

def ask_llm(weather, user_query):
    print("\n>>> Generowanie odpowiedzi...")
    
    context_text = (f"Weather in {weather['city']}: "
                    f"Temperature {weather['temp']} C, "
                    f"Condition {weather['condition']}, "
                    f"Wind {weather['wind_speed']} km/h.")

    full_prompt = (f"Context: {context_text}\n"
                   f"User Question: {user_query}\n"
                   f"Instruction: Answer short and helpful based on context.")

    try:
        payload = {"model": "tinyllama", "prompt": full_prompt, "stream": False}
        response = requests.post("http://ollama-service.default.svc:80/api/generate", json=payload, timeout=90)
        return response.json().get('response', 'Blad odpowiedzi modelu.')
    except Exception as e:
        return f"Blad polaczenia: {e}"

# --- GŁÓWNA PĘTLA PROGRAMU ---
while True:
    print("\n" + "="*40)
    print(" DOSTEPNE MIASTA:")
    mid = len(CITIES) // 2
    for i in range(mid):
        col1 = f"{i+1}. {CITIES[i]}"
        col2 = f"{i+1+mid}. {CITIES[i+mid]}"
        print(f"{col1:<20} {col2}")
    print("0. WYJSCIE Z PROGRAMU")
    print("="*40)

    try:
        raw_choice = input(f"Wybierz miasto (1-{len(CITIES)}) lub 0: ")
        choice = int(raw_choice)
        
        if choice == 0:
            print("Konczenie pracy...")
            break
            
        if 1 <= choice <= len(CITIES):
            selected_city = CITIES[choice - 1]
            
            # 1. Pobieramy pogode
            weather = get_weather_for_city(selected_city)
            
            print(f"\nPOGODA: {weather['city']} | {weather['temp']} C | {weather['condition']}")
            
            # 2. Pętla czatu (mozna zadawac wiele pytan do jednego miasta)
            while True:
                user_query = input(f"\nTwoje pytanie dotyczace {selected_city} (lub wpisz 'zmien' / 'exit'): ")
                
                if user_query.lower() in ['zmien miasto', 'back', 'wroc']:
                    break # Wracamy do wyboru miasta
                if user_query.lower() in ['exit', 'quit', 'wyjdz']:
                    print("Do widzenia!")
                    sys.exit(0) # Konczy caly program

                # Zapytanie do AI
                answer = ask_llm(weather, user_query)
                
                print("-" * 40)
                print(f"AI: {answer.strip()}")
                print("-" * 40)

        else:
            print("Nieprawidlowy numer.")
            
    except ValueError:
        print("Prosze wpisac liczbe.")
EOF
```

1. Uruchom skrypt:
    
    ```bash
    python test_rag_full.py
    ```
Przykład działania aplikacji:
<img width="1021" height="485" alt="image" src="https://github.com/user-attachments/assets/93481ae8-6dbe-49c5-9861-1fe34206e152" />
