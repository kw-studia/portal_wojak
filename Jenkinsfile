// Potok portal_wojak – wdrożenie portalu WojakNet (zespół WWW)

pipeline {
    agent { label 'agent_wojak' }

    parameters {
        choice(name: 'SRODOWISKO', choices: ['TEST', 'PROD'],
               description: 'TEST – nowa paczka z gałęzi main wdrażana na TEST; PROD – promocja paczki z budowania WERSJA')
        string(name: 'WERSJA', defaultValue: '',
               description: 'Numer budowania, którego paczka ma zostać wdrożona na PROD (dla TEST pomijany)')
    }

    triggers {
        // Sprawdzanie gałęzi main co ok. 5 minut – nowy commit uruchamia ścieżkę TEST
        pollSCM('H/5 * * * *')
    }

    options {
        disableConcurrentBuilds()    // blokada jednoczesnych przebiegów (i wdrożeń)
        skipDefaultCheckout()        // kod pobiera jawny etap „Pobranie kodu”
        timestamps()
        copyArtifactPermission('portal_wojak')   // zgoda na kopiowanie paczek tego zadania (ścieżka PROD)
    }

    environment {
        IP_TEST = '192.168.56.53'
        IP_PROD = '192.168.56.54'
        PACZKA  = "portal_wojak-b${env.BUILD_NUMBER}.tar.gz"
    }

    stages {
        stage('Pobranie kodu') {
            steps {
                checkout scm
                script {
                    env.COMMIT = sh(returnStdout: true, script: 'git rev-parse --short=7 HEAD').trim()
                }
                echo "Pobrano gałąź main, commit ${env.COMMIT}"
            }
        }

        stage('Walidacja') {
            when { expression { params.SRODOWISKO != 'PROD' } }
            steps {
                sh 'ansible-playbook --syntax-check wdrozenie_portalu.yml'
                sh 'ansible-lint --offline'
            }
        }

        stage('Przygotowanie paczki') {
            when { expression { params.SRODOWISKO != 'PROD' } }
            steps {
                script {
                    def data = sh(returnStdout: true, script: 'date -Iseconds').trim()
                    // version.json zawiera wyłącznie dane paczki (bez nazwy środowiska)
                    writeJSON file: 'strona/version.json', pretty: 2,
                              json: [build: env.BUILD_NUMBER, commit: env.COMMIT, data: data]
                }
                sh '''
                    cat strona/version.json
                    mkdir -p paczka
                    tar -czf "paczka/$PACZKA" strona wdrozenie_portalu.yml zadania inventory.yml ansible.cfg requirements.yml
                    tar -tzvf "paczka/$PACZKA"
                '''
            }
        }

        stage('Wdrożenie na TEST') {
            when { expression { params.SRODOWISKO != 'PROD' } }
            steps {
                rozpakujPaczke("paczka/${env.PACZKA}")
                wdrozPaczke('test')
            }
        }

        stage('Test na TEST') {
            when { expression { params.SRODOWISKO != 'PROD' } }
            steps {
                testPortalu("http://${env.IP_TEST}", 'TEST', env.BUILD_NUMBER)
            }
        }

        stage('Archiwizacja paczki') {
            when { expression { params.SRODOWISKO != 'PROD' } }
            steps {
                // Wyłącznie po udanym teście – paczkę mają tylko budowania, które przeszły testy na TEST
                archiveArtifacts artifacts: "paczka/${env.PACZKA}", fingerprint: true
            }
        }

        stage('Pobranie paczki') {
            when { expression { params.SRODOWISKO == 'PROD' } }
            steps {
                script {
                    if (!(params.WERSJA ==~ /\d+/)) {
                        error("Parametr WERSJA musi być numerem budowania (podano: '${params.WERSJA}')")
                    }
                }
                sh 'rm -rf promocja'
                // Brak paczki (budowanie nieudane, nieistniejące lub z PROD) przerywa potok
                copyArtifacts(projectName: env.JOB_NAME, selector: specific(params.WERSJA),
                              filter: 'paczka/*.tar.gz', target: 'promocja', flatten: true,
                              fingerprintArtifacts: true)
                rozpakujPaczke("promocja/portal_wojak-b${params.WERSJA}.tar.gz")
                script {
                    def wersja = readJSON file: 'wdrozenie/strona/version.json'
                    env.PACZKA_COMMIT = wersja.commit
                    env.PACZKA_DATA = wersja.data
                }
                echo "Paczka z budowania #${params.WERSJA}: commit ${env.PACZKA_COMMIT}, zbudowana ${env.PACZKA_DATA}"
            }
        }

        stage('Zatwierdzenie') {
            when { expression { params.SRODOWISKO == 'PROD' } }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(message: "Wdrożyć na PROD paczkę z budowania #${params.WERSJA} " +
                                   "(commit ${env.PACZKA_COMMIT}, zbudowana ${env.PACZKA_DATA})?",
                          ok: 'Wdróż na PROD')
                }
            }
        }

        stage('Wdrożenie na PROD') {
            when { expression { params.SRODOWISKO == 'PROD' } }
            steps {
                // Ta sama paczka, która przeszła testy na TEST – bez ponownego budowania
                wdrozPaczke('prod')
            }
        }

        stage('Test na PROD') {
            when { expression { params.SRODOWISKO == 'PROD' } }
            steps {
                testPortalu("http://${env.IP_PROD}", 'PROD', params.WERSJA)
            }
        }
    }

    post {
        failure {
            mail(to: env.EMAIL_POWIADOMIEN,
                 subject: "[Jenkins] Niepowodzenie: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """Budowanie zakończyło się niepowodzeniem.

Zadanie: ${env.JOB_NAME}
Numer budowania: ${env.BUILD_NUMBER}
Środowisko: ${params.SRODOWISKO}
Parametr WERSJA: ${params.WERSJA ?: '-'}
Logi: ${env.BUILD_URL}console
""")
        }
        cleanup {
            cleanWs()
        }
    }
}

// Rozpakowanie paczki do katalogu wdrozenie/ – wdrażana jest zawartość paczki, a nie katalog roboczy
def rozpakujPaczke(String plik) {
    sh """
        rm -rf wdrozenie
        mkdir wdrozenie
        tar -xzf '${plik}' -C wdrozenie
    """
}

// Wdrożenie wtyczką Ansible; klucz SSH konta deploy pochodzi z Jenkins Credentials (deploy-ssh)
def wdrozPaczke(String grupa) {
    dir('wdrozenie') {
        ansiblePlaybook(
            installation: 'ansible',
            playbook: 'wdrozenie_portalu.yml',
            inventory: 'inventory.yml',
            limit: grupa,
            credentialsId: 'deploy-ssh'
        )
    }
}

// Test po wdrożeniu wg wspólnych kryteriów testów; 5 prób, bo kontener może się jeszcze uruchamiać
def testPortalu(String adres, String srodowisko, String build) {
    retry(5) {
        try {
            def kod = sh(returnStdout: true, script: "curl -s -o /dev/null -w '%{http_code}' ${adres}/").trim()
            if (kod != '200') {
                error("GET ${adres}/ zwrócił kod ${kod} zamiast 200")
            }
            def tresc = sh(returnStdout: true, script: "curl -fsS ${adres}/")
            if (!tresc.contains('Wojak')) {
                error("GET ${adres}/ – treść nie zawiera nazwiska Wojak")
            }
            def health = sh(returnStdout: true, script: "curl -fsS ${adres}/health").trim()
            if (health != 'OK') {
                error("GET ${adres}/health zwrócił '${health}' zamiast OK")
            }
            def wersja = readJSON(text: sh(returnStdout: true, script: "curl -fsS ${adres}/version.json"))
            if (wersja.build.toString() != build) {
                error("version.json: build = ${wersja.build}, oczekiwano ${build}")
            }
            def srod = readJSON(text: sh(returnStdout: true, script: "curl -fsS ${adres}/srodowisko.json"))
            if (srod.srodowisko != srodowisko) {
                error("srodowisko.json: ${srod.srodowisko}, oczekiwano ${srodowisko}")
            }
            echo "Test ${srodowisko} OK: GET / = 200 (zawiera nazwisko), /health = OK, " +
                 "version.json build = ${wersja.build} (commit ${wersja.commit}), srodowisko.json = ${srod.srodowisko}"
        } catch (err) {
            echo "Test ${srodowisko} nieudany: ${err.message} – ponowienie za 5 s"
            sleep(time: 5, unit: 'SECONDS')
            throw err
        }
    }
}