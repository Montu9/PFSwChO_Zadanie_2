## Kuziora Karol 95468 - Zadanie 2





## Opis stacka
Implementowana aplikacja jest projektem zrealizowanym w trakcie poprzednich semestrów. Nosi ona nazwę Roommate i w założeniach miała służyć do zarządzania budżetem domowym przez wielu współlokatorów. Aplikacja powstała w oparciu o MERN stack.

- Klient - oparty o bibliotekę React
- Serwer - NodeJS oraz ExpressJS
- Baza danych - MongoDB

## Wolumeny
### MongoDB
Wdrożenie MongoDB zostało zrealizowane z wykorzystaniem StatefulSet, co gwarantuje utrzymanie spójności i trwałości danych w przypadku ponownego uruchomienia lub przenoszenia podów. Dodano również mechanizmach uwierzytelniania i autoryzacji, gdzie hasla użytkowników są zabezpieczone w Kubernetes Secrets.

Konfiguracja systemu bazuje na komponentach: ConfigMap do zarządzania podstawowymi ustawieniami oraz Secrets do przechowywania wrażliwych danych. 
W architekturze rozwiązania zastosowano usługę mongo-headless - specjalny typ usługi Kubernetes bez przypisanego adresu IP klastra. Jej głównym zadaniem jest zapewnienie bezpośredniej komunikacji między instancjami MongoDB w obrębie klastra. Kubernetes generuje rekordy DNS dla każdej repliki, co umożliwia bezpośrednie adresowanie poszczególnych węzłów bazy danych, jednocześnie uniemożliwiając dostęp z zewnątrz.

Dla każdej repliki bazy danych Kubernetes tworzy dedykowany rekord DNS, co pozwala na precyzyjne adresowanie poszczególnych instancji przy jednoczesnym zablokowaniu dostępu z zewnątrz klastra. Persystencja danych jest realizowana poprzez wolumeny z dostępem ReadWriteOnce, wykorzystujące standardową klasę pamięci Kubernetes.

Architektura przewiduje wdrożenie trzech replik bazy danych działających jako spójny zestaw. Każda replika posiada unikalny, stały identyfikator, który pozostaje niezmienny nawet po ponownym uruchomieniu lub przeniesieniu na inny węzeł klastra.

Zasoby obliczeniowe. Dla każdej repliki MongoDB zdefiniowano wymagania:
 - Minimalne zasoby: 256 MB pamięci, 200m mocy procesora
 - Maksymalne zasoby: 512 MB pamięci, 500m mocy procesora

Wdrożenie bazuje na zestawie replik (ReplicaSet) złożonym z trzech węzłów MongoDB. Inicjalizacja zestawu replik odbywa się na węźle głównym (PRIMARY) przy użyciu następującej komendy:

```
rs.initiate({ 
    _id: "rs0", 
    members: [ 
        { _id: 0, host: "mongo-0.mongo-headless.roommate.svc.cluster.local:27017" }, 
        { _id: 1, host: "mongo-1.mongo-headless.roommate.svc.cluster.local:27017" }, 
        { _id: 2, host: "mongo-2.mongo-headless.roommate.svc.cluster.local:27017" } 
    ] 
})
```

### Sekrety i ConfigMap
MongoDB oraz aplikacja Roommate korzystają z Kubernetes Secrets i ConfigMaps do zarządzania wrażliwymi danymi oraz konfiguracją. ConfigMaps, w przeciwieństwie do Secrets są wykorzystywane do przechowywania danych konfiguracyjnych, które nie są wrażliwe i mogą być bezpiecznie przechowywane w jawnej postaci.

```
apiVersion: v1
kind: Secret
metadata:
    name: backend-secrets
    namespace: roommate
type: Opaque
stringData:
    DATABASE_URI: "mongodb://user:SecretPassword@mongo-0.mongo-headless.roommate.svc.cluster.local:27017/Roommate?retryWrites=true&w=majority"
    ACCESS_TOKEN_SECRET: "542 (...) db97013636591fc44620b9f5ab17028dee1d2818d55"
    REFRESH_TOKEN_SECRET: "eb0b9f (...) 94d532a0"

```

## Klient oraz Serwer
### Frontend
Aplikacja wykorzystuje obraz montulll/clientr8:latest hostowany w publicznym repozytorium Docker Hub, co upraszcza proces wdrożenia i aktualizacji. W podobny sposób został opracowany serwer. Wykorzystując Dockerfile, dwa obrazy dotyczące tych dwóch mikroserwisów zostały zbudowane a następnie zpushowane do mojego repozytorium.

Zastosowana strategia RollingUpdate gwarantuje bezprzerwowe działanie aplikacji podczas aktualizacji. Parametr maxSurge: 1 pozwala na utworzenie jednej dodatkowej repliki podczas aktualizacji, podczas gdy maxUnavailable: 0 zapewnia, że wszystkie repliki pozostaną aktywne w trakcie tego procesu.

Określono precyzyjne limity zasobów dla kontenera:
```
resources:
    requests:
        memory: "256Mi"
        cpu: "100m"
    limits:
        memory: "512Mi"
        cpu: "300m"
```

Frontend jest dostępny poprzez usługę typu ClusterIP, która zapewnia wewnętrzną komunikację w klastrze:
```
kind: Service
metadata:
    name: roommate-frontend-service
spec:
    type: ClusterIP
    ports:
        - port: 3000
          targetPort: 3000
```
Podobnie jak w przypadku Mongo tutaj również zostały wykorzystane Secrets dla określenia adresu backendu, wykorzystywanego przez klienta do fetchowania danych:
```
apiVersion: v1
kind: Secret
metadata:
    name: frontend-secrets
    namespace: roommate
type: Opaque
stringData:
    API_BASE_URL: "http://roommate.local/api"
```

Aplikacja korzysta (HPA) dla automatycznego skalowania w zależności od obciążenia:

Minimalna liczba replik: 1
Maksymalna liczba replik: 3
Progi skalowania:
 - CPU: skalowanie przy średnim wykorzystaniu 70%
 - Pamięć: skalowanie przy średnim wykorzystaniu 80%

### Backend
Backend, działający na porcie 3001, jest chroniony przed bezpośrednim dostępem zewnętrznym i komunikuje się z bazą danych MongoDB poprzez wewnętrzną sieć klastra. 

Aplikacja NodeJs została skonfigurowana podobnie jak klient. Również posiada strategię RollingUpdate, wykorzystanie Secrets dla określenia zmiennych środowiskowych, podlega pod HPA.

### Ingress
Konfiguracja Ingress w aplikacji Roommate zapewnia zarządzanie ruchem HTTP i kompleksową obsługę komunikacji między komponentami systemu.

Zarządzanie routingiem:
 - Główna domena (roommate.local) obsługuje ruch do frontendu
 - Ścieżka /api kieruje żądania do backendu
 - Wykorzystanie wyrażeń regularnych umożliwia elastyczne dopasowywanie ścieżek

```
spec:
    rules:
        - host: roommate.local
          http:
              paths:
                  - path: /(.*)
                  - path: /api/?(.*)
```

Dalsza charakterystyka:
 - Zasoby są izolowane w przestrzeni nazw 'roommate'
 - Ruch przychodzący jest filtrowany i kierowany tylko do określonych serwisów
 - Serwis mongo nie jest bezpośrednio dostępny z zewnątrz

# Wdrożenie
Utworzenie przestrzeni nazw za pomocą kubectl create namespace roommate pozwala na logiczne wydzielenie i izolację wszystkich zasobów aplikacji. W jej zakresie utworzona została konfiguracja Resource Quota okreslająca ograniczenia obliczeniowe: 
```
kind: ResourceQuota
metadata:
    name: roommate-quota
    namespace: roommate
spec:
    hard:
        requests.cpu: "2"
        requests.memory: 2Gi
        limits.cpu: "4"
        limits.memory: 4Gi
        pods: "10"
```

Routing lokalny wymaga modyfikacji pliku hosts, co pozwala na przekierowanie żądań do lokalnego klastra Kubernetes. W moim przypadku korzystam z WSL działającego pod Windowsem. Dodanie wpisu 127.0.0.1 roommate.local umożliwia dostęp do aplikacji za pośrednictwem przyjaznej nazwy domeny. W tym przypadku dostęp z przeglądarki zostanie uzyskany za pomocą url "roommate.local"
Istotnym elementem jest uruchomienie tunelu sieciowego poprzez `minikube tunnel`. Tworzy  routing sieciowy,  co pozwala na poprawne funkcjonowanie usługi Ingress.

Inicjalizacja repliki MongoDB wymaga ręcznego wykonania komendy inicjalizacyjnej wewnątrz pierwszego kontenera bazy danych co zostało zaprezentowane w części sprawozdania poświęconej wolumenom.

# Uruchomienie
Uruchoemienie dodatku Ingress
`minikube addons enable ingress`

Wdrożenie komponentów z wykorzystaniem yaml
```
kubectl apply -f namespace-config.yaml
kubectl apply -f secrets.yaml
kubectl apply -f configmaps.yaml
kubectl apply -f database.yaml
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
kubectl apply -f autoskalering.yaml
```

Dodanie do /etc/host na wsl oraz windows
`127.0.0.1 roommate.local`

Uruchomienie tunelu
`minikube tunnel`

Kontrola wstawania podów 
`kubectl get pods -n roommate  -w`

## Zasoby Kubernetes'a
#### Namespace
   - roommate

#### ResourceQuota
   - roommate-quota

#### StatefulSet
   - mongo

#### Services
   - mongo-headless
   - roommate-backend-service
   - roommate-frontend-service

#### Deployments
   - roommate-backend
   - roommate-frontend

####. HorizontalPodAutoscaler
   - roommate-frontend-hpa
   - roommate-backend-hpa

#### ConfigMaps
   - mongo-init
   - mongo-config

#### Secrets
   - mongo-secrets
   - backend-secrets
   - frontend-secrets

#### Ingress
   - roommate-ingress

#### PersistentVolumeClaim
    - mongo-data