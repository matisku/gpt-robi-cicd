# **Automatyczny Deployment na AWS S3 przy użyciu GitHub Actions**

> **Prompt:**
> 
> Napisz krok po kroku jak używając GitHub Action zrobić deployment kodu na AWS.
> Opisz wszystkie wymagania, uprawnienia etc. Wygeneruj przykładową stronę html z prostym kodem javascript.

Rozszerzona wersja instrukcji obejmuje:

✅ Tworzenie pliku HTML

✅ Deployment na S3

✅ Konfigurację publicznego dostępu

**1\. Wymagania wstępne**
-------------------------

Przed rozpoczęciem:

1.  **Repozytorium na GitHub**, w którym będzie kod strony.
    
2.  **Bucket S3**, który posłuży jako hosting.
    
3.  **Dostęp IAM** z odpowiednimi uprawnieniami.
    
4.  **Sekrety w GitHub** (przechowujące klucze dostępowe).

**2\. Tworzenie Bucketa S3 i konfiguracja publicznego dostępu**
---------------------------------------------------------------

1.  **AWS Console → S3 → Create Bucket**
    
2.  **Ustawienia Bucketa**:
    
    *   **Bucket name:** your-bucket-name
        
    *   **Region:** np. us-east-1
        
    *   **Block all public access:** **Odznacz** wszystkie opcje
        
    *   **Potwierdź wyłączenie blokady dostępu publicznego**
        
3.  **Przejdź do zakładki "Permissions" → Bucket Policy**

Wklej następującą politykę (zmieniając your-bucket-name na właściwą nazwę):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```
4. **Ustawienia statycznego hostingu:**

*   Przejdź do zakładki **"Properties"**
    
*   W sekcji **"Static website hosting"** wybierz **"Enable"**
    
*   Ustaw **"Index document"** na index.html
    
*   Skopiuj **Endpoint URL**, np.:
```bash
http://your-bucket-name.s3-website-us-east-1.amazonaws.com
```
> [!CAUTION]
> **Upublicznienie w ten sposób strony narazi ją na ataki DDoS!**
> 
> Przy założeniu ze kod uruchomieniowy to zwyły test zatem:
>
> * Strona waży 1kb
>
> * AWS S3 ma wbudowany throtling po 5500req/s
>
> * Zakładamy że ktoś przez miesiac robi DDoS na stronę S3
>
> Po miesiącu takiego ataku koszt waszego bucketa wynosiłby **~$1,200**


**3\. Konfiguracja AWS IAM (Uprawnienia)**
------------------------------------------

Musimy utworzyć użytkownika IAM z odpowiednimi uprawnieniami do S3.

1.  **AWS Console → IAM → Users → Add User**
    
2.  **Nadaj dostęp programowy (Programmatic access)**
    
3.  **Przypisz uprawnienia**Stwórz **politykę IAM** z następującą treścią:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```
4. **Zapisz Access Key ID i Secret Access Key**
   
**4\. Dodanie Sekretów do GitHub**
----------------------------------

Przejdź do **GitHub → Settings → Secrets and variables → Actions → New repository secret**Dodaj:

*   `AWS_ACCESS_KEY_ID`
    
*   `AWS_SECRET_ACCESS_KEY`
    
*   `AWS_REGION` → np. `eu-west-1`
    
*   `S3_BUCKET_NAME` → `your-bucket-name`

> [!CAUTION]
> Nawet [AWS sugeruje](https://docs.aws.amazon.com/cli/v1/userguide/cli-chap-authentication.html) żeby nie używać tej metody autentykacji. 
> ![image](/img/aws-auth.png)
> Zamiast tego można użyć dedykowanej roli z użyciem [GitHub jako Identity Provider (IdP) w AWS IAM](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/). 

**5\. Tworzenie workflow GitHub Actions**
-----------------------------------------

Utwórz plik `.github/workflows/deploy.yml`:
```yaml
name: Deploy to S3

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy to S3
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} --delete --acl public-read
```
> [!CAUTION]
> Pojawia się tutaj klika problemów:
> * Parametr ```--acl public-read``` spowoduje że ta akcja nie zadziała, poniewa nigdzie wcześniej nie było ustawione ACL na S3 bucket
> * Oczywiście używamy tutaj kluczy, a klucze maja to do siebie że nie rotowane lubią wyciekać
> * Parametr ```sync .``` spowoduje że cały bucket będzie synchronizowany na S3, a raczej tego nie chcemy
>   
> ![bucket-struct](/img/bucket-struct.png)

**6\. Sprawdzenie poprawności wdrożenia**
-----------------------------------------

Po uruchomieniu workflow, sprawdź dostępność strony:

1.  Otwórz http://your-bucket-name.s3-website-eu-west-1.amazonaws.com
    
2.  Powinna się wyświetlić Twoja strona HTML! 🎉
    

**Podsumowanie**
----------------

🚀 **Finalne kroki dla automatycznego hostingu na S3:**

✅ **Tworzenie pliku HTML**

✅ **Konfiguracja publicznego dostępu w S3**

✅ **Konfiguracja IAM + GitHub Secrets**

✅ **Automatyczne wdrażanie z GitHub Actions**

Dzięki temu masz **pełny pipeline CI/CD** dla statycznej strony na AWS S3! 🚀

--------------------------------------------------------------------------------
> [!TIP]
> **Wnioski**:
> * Vibe Coding jest fajny
> * ... ale wymaga jednak sporo uwagi
> * Ważne jest żeby wiedzieć co chce się osiągnać
> * ... i być swiadomym swojej niewiedzy
> * a za 10 lat, będziemy pracować jako **Senior AI Technical Debt Engineer** :grin:
