---
title: "Forcer l'UTC dans Doctrine sans toucher aux entités"
date: 2017-02-19
categories: [développement]
tags: [php, symfony, doctrine, datetime, utc]
description: "Comment surcharger les types natifs de Doctrine pour imposer l'UTC partout, sans modifier une seule entité."
---

Un timestamp qui revient de la base de données avec une heure de décalage. Pas à chaque fois. Uniquement quand le serveur de dev tourne en `Europe/Paris` et que la CI tourne en UTC. Le genre de bug qui disparaît quand on le cherche et qui revient en production un vendredi soir.

Le problème n'est pas dans la logique métier. Il est dans ce que Doctrine fait discrètement avec les dates.

## Ce que Doctrine fait par défaut

Quand on déclare un champ `datetime` dans une entité Doctrine, la conversion entre PHP et la base de données passe par `DateTimeType`. Cette classe appelle `format()` sur l'objet `DateTime` pour écrire en base, et `DateTime::createFromFormat()` pour le relire. Aucune mention de timezone nulle part.

Si l'objet PHP est en `Europe/Paris`, Doctrine formate `2017-01-15 11:30:00` et l'écrit tel quel. Si le serveur qui lit ce champ est en UTC, il obtient `2017-01-15 11:30:00` et l'interprète comme UTC. Une heure s'est évaporée dans l'aller-retour, sans le moindre message d'erreur.

<a href="https://www.doctrine-project.org/projects/doctrine-orm/en/latest/cookbook/working-with-datetime.html" target="_blank" rel="noopener noreferrer">La doc Doctrine couvre ce sujet</a> et suggère des types personnalisés comme solution. Ce qu'elle mentionne en passant, c'est qu'on peut donner à ces types personnalisés le même nom que les types natifs. Ce détail change tout.

## Remplacer, pas ajouter

La plupart des exemples de types Doctrine personnalisés introduisent un nouveau nom : `utc_datetime`, `app_date`, et ainsi de suite. On annote ensuite chaque champ avec `type: 'utc_datetime'` dans les entités. Ça fonctionne, mais c'est fastidieux et ça ne protège pas contre un `type: 'datetime'` oublié.

L'autre option : enregistrer le type personnalisé sous le nom `datetime`. Doctrine remplace sa propre implémentation par la nôtre, partout, sans exception. Chaque champ `datetime` de toutes les entités passe par notre logique, sans changer une seule annotation.

C'est ce qu'on vient de déployer sur notre plateforme de microservices PHP. Voici à quoi ça ressemble.

## Le trait partagé

Les deux types (`date` et `datetime`) partagent la même logique de conversion via un trait :

```php
trait UTCDate
{
    private \DateTimeZone $utc;

    public function convertToPHPValue($value, AbstractPlatform $platform): ?\DateTime
    {
        if (null === $value || $value instanceof \DateTime) {
            return $value;
        }

        $format = $this->getFormat($platform);
        $converted = \DateTime::createFromFormat($format, $value, $this->getUtc());

        if (!$converted) {
            throw new \RuntimeException(
                sprintf('Could not convert database value "%s" to DateTime using format "%s".', $value, $format)
            );
        }

        $this->postConvert($converted);

        return $converted;
    }

    abstract protected function getFormat(AbstractPlatform $platform): string;

    private function getUtc(): \DateTimeZone
    {
        if (empty($this->utc)) {
            $this->utc = new \DateTimeZone('UTC');
        }

        return $this->utc;
    }
}
```

Le point clé : `\DateTime::createFromFormat()` reçoit une timezone UTC explicite. La valeur brute issue de la base est interprétée comme UTC, peu importe la timezone configurée sur le serveur PHP.

## UTCDateTimeType

Pour les champs `datetime`, le chemin d'écriture doit aussi imposer l'UTC :

```php
class UTCDateTimeType extends DateTimeType
{
    use UTCDate;

    #[\Override]
    public function convertToPHPValue($value, AbstractPlatform $platform): ?\DateTime
    {
        if (null === $value || $value instanceof \DateTimeInterface) {
            return parent::convertToPHPValue($value, $platform);
        }

        return parent::convertToPHPValue("$value+0000", $platform);
    }

    #[\Override]
    public function convertToDatabaseValue($value, AbstractPlatform $platform): ?string
    {
        if ($value instanceof \DateTime) {
            $value->setTimezone($this->getUtc());
        }

        return parent::convertToDatabaseValue($value, $platform);
    }

    #[\Override]
    protected function getFormat(AbstractPlatform $platform): string
    {
        return $platform->getDateTimeFormatString();
    }

    protected function postConvert(\DateTime $converted): void {}
}
```

En lecture (`convertToPHPValue`), si la valeur est une chaîne brute, on y ajoute `+0000` avant de déléguer au parent. Le parent utilise ensuite ce suffixe de timezone pour créer correctement l'objet PHP.

En écriture (`convertToDatabaseValue`), on force le `DateTime` en UTC avant de le formater. Ce qui va en base est toujours en UTC.

## UTCDateType

Pour les colonnes `date` (sans composante horaire), même approche avec une étape supplémentaire :

```php
class UTCDateType extends DateType
{
    use UTCDate;

    #[\Override]
    protected function getFormat(AbstractPlatform $platform): string
    {
        return $platform->getDateFormatString();
    }

    protected function postConvert(\DateTime $converted): void
    {
        $converted->setTime(0, 0, 0);
    }
}
```

La méthode `postConvert()` remet l'heure à `00:00:00` après le parsing. Sans elle, un champ `date` pourrait revenir avec `23:59:59` ou `00:00:00+02:00` selon la timezone du serveur, ce qui casse les comparaisons et le tri.

## Enregistrement dans Symfony

La partie décisive : déclarer les types sous leurs noms natifs dans `config/packages/doctrine.yaml`.

```yaml
doctrine:
    dbal:
        types:
            date:
                class: App\Doctrine\DBAL\Types\UTCDateType
            datetime:
                class: App\Doctrine\DBAL\Types\UTCDateTimeType
```

C'est tout. Doctrine échange ses propres implémentations contre les nôtres. Les entités existantes ne changent pas, les migrations ne bougent pas, les annotations restent `type: Types::DATETIME_MUTABLE`. Le comportement change globalement, sans friction.

## 12 microservices, 89 colonnes, un bloc de config

Ces deux types tournent maintenant sur 12 microservices indépendants, chacun avec sa propre config Doctrine, couvrant 89 colonnes de production. Les serveurs CI tournent en UTC, les machines de dev en `Europe/Paris`, les données voyagent entre eux sans dériver. Ce n'est pas spectaculaire. C'est juste fiable.

La vraie leçon n'est pas technique : un problème de timezone non résolu est un problème d'intégrité des données. Les décalages s'accumulent silencieusement, les comparaisons se trompent, les exports deviennent inexacts. Deux lignes de config et trois classes peuvent prévenir ça définitivement.
