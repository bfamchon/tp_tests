# TP ARCHITECTURE

## Présentation du contexte

Vous développez une application de gestion de webinaires en suivant les concepts de l'architecture Ports / Adapters.

Deux `use-case` sont déjà implémentés : organiser un webinaire (`organize-webinar`) et changer le nombre de place disponibles dans un webinaire (`change-seat`).

Dans le précédent TP, vous avez peut-être réalisé les tests unitaires pour couvrir la fonctionnalité demandée, `book-seat`.

Cette fois-ci, nous allons implémenter des tests unitaires et E2E sur la fonctionalité `change-seats`, ainsi que des tests d'intégration sur un repository `mongo-webinar-repository`.

Cette approche vous apportera une autre vision de comment s'y prendre pour tester unitairement vos fonctionnalités !

Pour cette fonctionnalité `change-seat`, voici quelques règles métier :

- seul l'organisateur peut changer le nombre de siège disponible
- nous ne pouvons pas revoir un nombre de siège à la baisse

## Marche à suivre

### Création d'un fichier de test unitaire

Nous allons commencer par créer un fichier `change-seat.test.ts` sur lequel vous allez implémenter les tests unitaires de la fonctionnalité `change-seat`.

Pour organiser un test unitaire, intéréssons nous d'abord à la structure d'un test.

Au premier describe, indiquez la fonctionnalité testée.

```typescript
describe('Feature : Change seats', () => {
  ...
});
```

Au second, le scénario au sein de cette fonctionnalité.

```typescript
describe('Feature : Change seats', () => {
  ...// Initialisation de nos tests, boilerplates...
  describe('Scenario: Happy path', () => {
    ...
  });
});
```

Puis terminer par la règle métier qui est testée.

```typescript
describe('Feature : Change seats', () => {
  ...// Initialisation de nos tests, boilerplates...
  describe('Scenario: Happy path', () => {
    ...// Code commun à notre scénario : payload...
    it('should change the number of seats for a webinar', async () => {
      ...// Vérification de la règle métier, condition testée...
    });
  });
});
```

Nous pouvons maintenant écrire notre premier test, en commençant la payload que nous allons utiliser, soit une demande de l'utilisateur `alice` de changer le nombre de place à `200` pour le webinaire `webinar-id` :

```typescript
   describe('Scenario: happy path', () => {
    const payload = {
      user: testUser.alice,
      webinarId: 'webinar-id',
      seats: 200,
    };
    ...
  });
```

Nous cherchons maintenant à :

- executer notre use-case
- tester que le scénario passe comme attendu

Place à l'écriture :

```typescript
it('should change the number of seats for a webinar', async () => {
  // ACT
  await useCase.execute(payload);
  // ASSERT
  const updatedWebinar = await webinarRepository.findById('webinar-id');
  expect(updatedWebinar?.props.seats).toEqual(200);
});
```

Vous remarquerez sans doute qu'il manque quelques items avant de pouvoir faire passer notre test unitaire, le `useCase` n'est pas défini, ni le `webinarRepository`.

Toutes ces déclarations vont se faire sous le premier `describe`, et ce sont souvent les étapes que vous allez devoir faire peu importe la règle métier testée.

Nous cherchons donc à faire :

- initialiser le use-case
- initialiser un repository
- populer un webinaire dans ce repository, pour que l'on puisse appliquer les règles métier et vérifier
- avant chaque test, repartir d'un état initial, pour garantir l'indépendance entre plusieurs éxecutions.

Allons y :

```typescript
describe('Change seats', () => {
    let webinarRepository: InMemoryWebinarRepository;
    let useCase: ChangeSeats;

    const webinar = new Webinar({
        id: 'webinar-id',
        organizerId: testUser.alice.props.id,
        title: 'Webinar title',
        startDate: new Date('2024-01-01T00:00:00Z'),
        endDate: new Date('2024-01-01T01:00:00Z'),
        seats: 100,
    });

    beforeEach(() => {
        webinarRepository = new InMemoryWebinarRepository([webinar]);
        useCase = new ChangeSeats(webinarRepository);
    });
  ...
});
```

Le premier scénario devrait passer au vert !

Une écriture du test unitaire après le code n'est pas forcément le + adapté car vous partez avec un esprit biaisé.

D'ailleurs, on pourrait ne pas y voir d'intérêt...

Et vous avez raison !

Pour l'écriture des tests unitaires, il est beaucoup + agréable de fonctionner en TDD, développement piloté par les tests, ou en test first, selon la complexité de ce qu'on développe.

De ce fait, on ne développe que le code nécessaire, naturellement couvert par un test, et le design émerge petit à petit avec les refactoring.

Le but du TP n'était pas de vous faire écrire du code business, mais comment on aurait pu s'y prendre ?

Voici quelques recommandations :

- je commence toujours par écrire mon premier scénario de test et le résultat attendu
- je déclare ensuite les différentes variables que je vais devoir utiliser
- je code les implémentations et retourne un premier résultat pour satisfaire mon test et le faire passer au vert
- j'entame une phase de refactoring pour rendre mon test et mon code + élégant.
- je passe au scénario suivant

Passons au scénario suivant, que ce passe-t-il si le webinaire n'existe pas ?

```typescript
describe('Scenario: webinar does not exist', () => {
    const payload = {
      ...
    };
    it('should fail', async () => {
      ...
    });
});
```

À vous de jouer : Quelle serait ma payload cette fois-ci, pour que l'on vérifie que le scénario est bien couvert ?

À vous de jouer : Quel serait le test à écrire pour vérifier que le bon message d'erreur a été lancé ?

Un indice : `await expect(useCase.execute(payload)).rejects.toThrow("mon message d'erreur");`

Bon, c'était relativement simple...

Il ne faut pas oublier de vérifier que le webinaire initial n'a pas été modifié !

Ajoutons donc le code suivant :

```typescript
const webinar = webinarRepository.findByIdSync('webinar-id');
expect(webinar?.props.seats).toEqual(100);
```

Pour la suite, ce sera à votre tour. Voici ce qu'il nous reste à vérifier :

- Scenario: update the webinar of someone else
- Scenario: change seat to an inferior number
- Scenario: change seat to a number > 1000

Vous trouverez tout ce qu'il vous faut en regardant le use-case.

Une fois que les différents scénarios sont couverts, il ne faut pas hésiter à faire un refactoring global, du code et des tests !

Si vous êtes arrivé jusque ici, vous avez peut-être remarqué que certaines étapes sont répétées souvent, par exemple :

```typescript
const webinar = webinarRepository.findByIdSync('webinar-id');
expect(webinar?.props.seats).toEqual(100);
```

Pour vérifier que le webinaire reste inchangé... Faisons donc une méthode partagé, sous le premier `describe`, qui soit un peu + parlante !

```typescript
...
  function expectWebinarToRemainUnchanged() {
    const webinar = webinarRepository.findByIdSync('webinar-id');
    expect(webinar?.props.seats).toEqual(100);
  }
...
```

Il ne faut pas hésiter à rendre nos tests le + parlant possible, ils font alors l'objet d'une documentation vivante que n'importe qui peut comprendre en lisant.

On pourrait remplacer notre `await useCase.execute(payload);` par une méthode `await whenUserChangeSeatsWith(payload)`

Et :

```typescript
const updatedWebinar = await webinarRepository.findById('webinar-id');
expect(updatedWebinar?.props.seats).toEqual(200);
```

Par `thenUpdatedWebinarSeatsShouldBe(200)`

Grâce à ces méthodes, nous construisons petit à petit ce que l'on appel des fixtures.

Ces méthodes pourraient également être exportées du test pour ne laisser que du verbal... Qui a dit que l'écriture de tests était chiant ?!

### Création d'un test d'intégration

Nous allons maintenant réaliser le premier test d'intégration, sur un nouveau repository.

Jusqu'à présent, nous avons travaillé avec un repository in-memory, très utile pour débuter dans la création de nos use-cases et dans nos tests unitaires.

Mais ça n'aurait pas vraiment de sens de faire un test d'intégration sur un in-memory...

Nous allons donc créer un repository qui va s'interfacer avec une vraie base de donnée, en utilisant [l'ORM Prisma](https://prisma.io).

Allons, y : créons deux fichiers `webinars/adapters/webinar-repository.prisma.ts` et `webinars/adapters/webinar-repository.prisma.int.test.ts`

```typescript
import { PrismaClient, Webinar as PrismaWebinar } from '@prisma/client';
import { Webinar } from 'src/webinars/entities/webinar.entity';
import { IWebinarRepository } from 'src/webinars/ports/webinar-repository.interface';

export class PrismaWebinarRepository implements IWebinarRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async create(webinar: Webinar): Promise<void> {
    await this.prisma.webinar.create({
      data: WebinarMapper.toPersistence(webinar),
    });
    return;
  }
  async findById(id: string): Promise<Webinar | null> {
    const maybeWebinar = await this.prisma.webinar.findUnique({
      where: { id },
    });
    if (!maybeWebinar) {
      return null;
    }
    return WebinarMapper.toEntity(maybeWebinar);
  }
  async update(webinar: Webinar): Promise<void> {
    await this.prisma.webinar.update({
      where: { id: webinar.props.id },
      data: WebinarMapper.toPersistence(webinar),
    });
    return;
  }
}

class WebinarMapper {
  static toEntity(webinar: PrismaWebinar): Webinar {
    return new Webinar({
      id: webinar.id,
      organizerId: webinar.organizerId,
      title: webinar.title,
      startDate: webinar.startDate,
      endDate: webinar.endDate,
      seats: webinar.seats,
    });
  }

  static toPersistence(webinar: Webinar): PrismaWebinar {
    return {
      id: webinar.props.id,
      organizerId: webinar.props.organizerId,
      title: webinar.props.title,
      startDate: webinar.props.startDate,
      endDate: webinar.props.endDate,
      seats: webinar.props.seats,
    };
  }
}
```

Dans le premier fichier, nous allons implémenter le `PrismaWebinarRepository` et un mapper, histoire de passer notre entité du domaine à l'infrastructure sereinement.

Rien de bien compliqué ici.

En production, l'application pourra maintenant utiliser un repository DB plutôt qu'in-memory en un minimum d'adaptation grâce à l'injection de dépendances : c'est la force d'une architecture propre !

Naturellement, nous allons avoir un peu plus de travail de mise en place côté testing. Car il faut maintenant démarrer une DB déidée, gérer les intérractions pour que nos tests restent indépendant...

Dans la pratique, on essaiera toujours de séparer l'éxecution de ces tests d'intégration (ou e2e), qui sont plus coûteux que les tests unitaires de par leur nature.

Les tests unitaires sont très bien pour le développement et le feedback instantané, les tests d'intégrations se lanceront plutôt à la fin d'un développement ou dans une chaîne d'intégration.

Un fichier de configuration et une tâche npm sont dédiés : `npm run test:int`.

Passons maintenant à lécriture du test, voici ce que l'on cherche à faire :

- déclarer tout ce dont on va avoir besoin
- démarrer une DB dédiée aux tests
- effectuer les migrations pour que la DB soit synchronisée
- nettoyer cette DB entre chaque tests pour garantir l'indépendance
- stopper la DB proprement après l'execution des tests

Pour ces opérations, nous allons utiliser `testcontainers`. C'est une librairie qui va nous permettre de démarrer n'importe quel service qui tournerai dans un container docker, pour réaliser simplement des tests.

Voici les variables dont nous allons avoir besoin :

```typescript
import { PrismaClient } from '@prisma/client';
import {
  PostgreSqlContainer,
  StartedPostgreSqlContainer,
} from '@testcontainers/postgresql';
import { exec } from 'child_process';
import { PrismaWebinarRepository } from 'src/webinars/adapters/webinar-repository.prisma';
import { Webinar } from 'src/webinars/entities/webinar.entity';
import { promisify } from 'util';
const asyncExec = promisify(exec);

describe('PrismaWebinarRepository', () => {
  let container: StartedPostgreSqlContainer;
  let prismaClient: PrismaClient;
  let repository: PrismaWebinarRepository;
  ...
```

Nous allons ensuite ajouter une action avant le lancement des tests, démarrer la DB et réaliser les migrations :

```typescript
beforeAll(async () => {
  container = await new PostgreSqlContainer()
    .withDatabase('test_db')
    .withUsername('user_test')
    .withPassword('password_test')
    .withExposedPorts(5432)
    .start();

  const dbUrl = container.getConnectionUri();
  prismaClient = new PrismaClient({
    datasources: {
      db: { url: dbUrl },
    },
  });

  await asyncExec(`DATABASE_URL=${dbUrl} npx prisma migrate deploy`);

  return prismaClient.$connect();
});
```

Ensuite, avant chaque test, on va s'assurer de ré-initialiser le repository et supprimer les nettoyer la DB :

```typescript
beforeEach(async () => {
  repository = new PrismaWebinarRepository(prismaClient);
  await prismaClient.webinar.deleteMany();
  await prismaClient.$executeRawUnsafe('DELETE FROM "Webinar" CASCADE');
});
```

Et pour finir, après le lancement de tous les tests, nous allons arrêter le container :

```typescript
afterAll(async () => {
  await container.stop({ timeout: 1000 });
  return prismaClient.$disconnect();
});
```

Un peu + de boilerplate inhérent à ce mode de tests... C'est aussi pour cette raison que l'on en fait naurellement moins, et pas avec la même approche.

En effet, ici, on adopetera plutôt une logique de "test first" ou "test last" plutôt que de faire du TDD car le feedback est relativement long.

Allons y pour la suite, nous allons tester chaque méthode de notre repository :

```typescript
describe('Scenario : repository.create', () => {
  it('should create a webinar', async () => {
    // ARRANGE
    const webinar = new Webinar({
      id: 'webinar-id',
      organizerId: 'organizer-id',
      title: 'Webinar title',
      startDate: new Date('2022-01-01T00:00:00Z'),
      endDate: new Date('2022-01-01T01:00:00Z'),
      seats: 100,
    });

    // ACT
    await repository.create(webinar);

    // ASSERT
    const maybeWebinar = await prismaClient.webinar.findUnique({
      where: { id: 'webinar-id' },
    });
    expect(maybeWebinar).toEqual({
      id: 'webinar-id',
      organizerId: 'organizer-id',
      title: 'Webinar title',
      startDate: new Date('2022-01-01T00:00:00Z'),
      endDate: new Date('2022-01-01T01:00:00Z'),
      seats: 100,
    });
  });
});
```

Si vous avez bien remarqué, on évite d'utiliser notre repository pour la partie ASSERT, on utilise ici directement le `prismaClient` pour isoler le test de la méthode `create`.

À vous de jouer ! Vous allez maintenant suivre la même logique pour tester le `findById` et le `update`.

## Astuces

La commande `npm run test:watch` pour lancer vos tests en watch mode.
