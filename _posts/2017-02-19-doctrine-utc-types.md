---
layout: post
title: "Enforcing UTC in Doctrine without touching your entities"
date: 2017-02-19
categories: [development]
tags: [php, symfony, doctrine, types]
description: "How to override Doctrine's built-in types to enforce UTC everywhere, without touching a single entity."
---

A timestamp coming back from the database one hour off. Not every time. Only when the dev server runs in `Europe/Paris` and CI runs in UTC. The kind of bug that disappears when you look for it and comes back in production on a Friday evening.

The problem isn't in the business logic. It's in what Doctrine quietly does with dates.

## What Doctrine does by default

When you declare a `datetime` field in a Doctrine entity, the conversion between PHP and the database goes through `DateTimeType`. That class calls `format()` on your `DateTime` object to write to the database, and `DateTime::createFromFormat()` to read it back. No mention of timezone anywhere.

If your PHP object is in `Europe/Paris`, Doctrine formats `2017-01-15 11:30:00` and writes it as-is. If the server reading that field is in UTC, it gets `2017-01-15 11:30:00` and interprets it as UTC. One hour has evaporated in the round trip, without a single error message.

<a href="https://www.doctrine-project.org/projects/doctrine-orm/en/latest/cookbook/working-with-datetime.html" target="_blank" rel="noopener noreferrer">The Doctrine docs cover this</a>, suggesting custom types as the fix. What they mention in passing is that you can give those custom types the same name as the built-in ones. That detail changes everything.

## Replace, don't add

Most custom Doctrine type examples introduce a new name: `utc_datetime`, `app_date`, and so on. You then annotate every field with `type: 'utc_datetime'` in the entities. It works, but it's tedious and doesn't protect against a forgotten `type: 'datetime'`.

The other option: register the custom type under the name `datetime`. Doctrine replaces its own type with yours, everywhere, no exceptions. Every `datetime` field across all entities goes through your logic, without changing a single annotation.

That's what we just shipped across our PHP microservices platform. Here's what it looks like.

## The shared trait

Both types (`date` and `datetime`) share the same conversion logic through a trait:

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

The key: `\DateTime::createFromFormat()` receives an explicit UTC timezone. The raw value from the database is interpreted as UTC, regardless of what the PHP server's timezone is set to.

## UTCDateTimeType

For `datetime` fields, the write path also needs to enforce UTC:

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

On read (`convertToPHPValue`), if the value is a raw string, we append `+0000` before delegating to the parent. The parent then uses that timezone suffix to create the PHP object correctly.

On write (`convertToDatabaseValue`), we force the `DateTime` to UTC before formatting. What goes into the database is always UTC.

## UTCDateType

For `date` columns (no time component), same approach with one extra step:

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

The `postConvert()` method resets the time to `00:00:00` after parsing. Without it, a `date` field might come back with `23:59:59` or `00:00:00+02:00` depending on the server's timezone, which breaks comparisons and ordering.

## Registering in Symfony

The decisive part: declaring the types under their built-in names in `config/packages/doctrine.yaml`.

```yaml
doctrine:
    dbal:
        types:
            date:
                class: App\Doctrine\DBAL\Types\UTCDateType
            datetime:
                class: App\Doctrine\DBAL\Types\UTCDateTimeType
```

That's it. Doctrine swaps out its own implementations for yours. Existing entities don't change, migrations don't move, annotations stay `type: Types::DATETIME_MUTABLE`. The behavior changes globally, without friction.

## 12 microservices, 89 columns, one config block

These two types are now running across 12 independent microservices, each with its own Doctrine config, covering 89 production columns. CI servers run in UTC, dev machines in `Europe/Paris`, data travels between them without shifting. It's not spectacular. It's just reliable.

The real lesson isn't technical: an unresolved timezone issue is a data integrity issue. Offsets accumulate silently, comparisons go wrong, exports become inaccurate. Two lines of config and three classes can prevent that permanently.
