# [Restore `Message` from Symfony email](https://github.com/yiisoft/yii2-symfonymailer/issues/72)

- _Type:_ Issue
- _State:_ closed
- _Repository:_ `yii2-symfonymailer`
- _Created at:_ 2024-04-05 13:54:15

### What steps will reproduce the problem?

```php
$originalMessage = Yii::$app->mailer->compose()
    ->setFrom('foo@bar.com')
    ->setTo('bar@foo.com')
    ->setTextBody('Test')
    ->attach('/path/to/attachment.pdf')
    ->setSubject('Test');

 $email = $originalMessage->getSymfonyEmail();

 $restoredMessage = new yii\symfonymailer\Message(['email' => $email]);
```

### What's expected?

A message can be succesfully restored from the source `Symfony\Component\Mime\Email` object.

### What do you get instead?

An `yii\base\UnknownPropertyException` with the message `Setting unknown property: yii\symfonymailer\Message::email`.

### What is the use-case?

An email queue that stores the message on disk. The Symfony email message can be serialized and stored on disk. However, when it is unserialized there is no way to restore the `yii\symfonymailer\Message` object and send it.

### Additional info

| Q                         | A
| ------------------------- | ---
| Yii version               | 2.0.49.3
| Yii SymfonyMailer version | 2.0.4
| SymfonyMailer version     | v6.4.4
| PHP version               | 8.1.27
| Operating system          | Debian

## Comments

skepticspriggan on 2024-04-06 10:19

> I have come to understand this is a non-issue since the `Message` object itself can also be serialized.

ItsReddi on 2024-06-19 07:54

> > I have come to understand this is a non-issue since the `Message` object itself can also be serialized.

Could you please explain a bit further how you solved this?
