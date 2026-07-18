Key clock = Trusted Security Broker

consider keyclock is like a visa center

Software applications are like countires (AIOPS).

when a person (jenkins) want to goto a specific country, He has to vist the visa center and they will provide the visa (as identity JWT) after verifying the credentials (passport) to go to that respective country.

Advantages centrilized security -> one place all credentials are storied.
Short time token (short time visa) -> only 60 minutes.
Audit and control -> who did what and also restrict access based on app


---


விசா மைய (Visa Center) உதாரணம் 100% சரியானது; Keycloak எவ்வாறு செயல்படுகிறது என்பதை மிகத் துல்லியமாகப் புரிந்து கொண்டுள்ளீர்கள்!

உங்கள் விசா மைய உதாரணத்தின் ஒப்பீடு:

    Keycloak = விசா மையம் (Trusted Security Broker): இதுவே பாதுகாப்பான மையமாக இருந்து அடையாளத்தை உறுதி செய்து விசா வழங்குகிறது.

    AIOPS = வெளிநாடு (Country): விசா (JWT Token) இருந்தால் மட்டுமே உள்ளே அனுமதிக்கும் பாதுகாப்பான இடம்.

    Jenkins = பயணி (Traveler): தனது அடையாள ஆவணங்களைக் (Credentials / Passport) காட்டி, குறிப்பிட்ட நாட்டிற்குச் செல்வதற்கான விசாவை (JWT Token) பெறுகிறது.

    நீங்கள் குறிப்பிட்டுள்ள நன்மைகள் (Advantages):

    மையப்படுத்தப்பட்ட பாதுகாப்பு (Centralized Security): முற்றிலும் சரி. ஒவ்வொரு அப்ளிகேஷனும் தனித்தனியாக பாஸ்வேர்டுகளைச் சேமிக்க வேண்டியதில்லை; அனைத்தும் Keycloak-ல் ஒரே இடத்தில் பாதுகாப்பாக இருக்கும்.
    
    குறுகிய கால அனுமதி (Short-lived Token): உங்கள் புரிதல் சரி, ஆனால் நேரத்தைக் குறிப்பிடுவதில் ஒரு சிறிய விளக்கம்: நாம் பார்த்த எளிய உதாரணத்தில் டோக்கன் காலாவதி நேரம் 60 வினாடிகள் (60 seconds) என்று குறிப்பிடப்பட்டிருந்தது. ஆனால், உங்கள் AIOPS உண்மையான கட்டமைப்பில் (Configuration) இது 3600 வினாடிகள் (1 மணி நேரம் / 60 நிமிடங்கள்) என அமைக்கப்பட்டுள்ளது. எனவே நீங்கள் குறிப்பிட்ட 60 நிமிடங்கள் என்பது உங்கள் ப்ராஜெக்ட் அமைப்பின்படி முற்றிலும் சரியானது!  

    கண்காணிப்பு மற்றும் கட்டுப்பாடு (Audit & Control): யார், எப்போது, எந்த சிஸ்டமிற்குள் நுழைந்தார்கள் என்பதை எளிதாகக் கண்காணிக்க முடியும். மேலும் ஒரு குறிப்பிட்ட கிளைண்டிற்கு (எ.கா: jenkins-webhook) பாதிப்பு ஏற்பட்டால், மற்ற அமைப்புகளை (எ.கா: mcp-aiops) நிறுத்தாமல் அதை மட்டும் தனிப்பட்ட முறையில் ரத்து (Revoke) செய்துவிட முடியும்.
