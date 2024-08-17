#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <ctime>

using namespace std;

class Card {
public:
	enum rank { ACE = 1, TWO, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT, NINE, TEN, JACK, QUEEN, KING };
	enum suit { CLUBS, DIAMONDS, HEARTS, SPADES };
	friend ostream& operator<<(ostream& os, const Card& aCard);
	Card(rank r = ACE, suit s = SPADES, bool IsFaceUp = true);
	int GetValue() const;
	void Flip();
private:
	rank m_Rank;
	suit m_Suit;
	bool m_IsFaceUp;
};

Card::Card(rank r, suit s, bool IsFaceUp) :
	m_IsFaceUp(IsFaceUp),
	m_Rank(r),
	m_Suit(s) {}

int Card::GetValue() const {
	int value = 0;
	if (m_IsFaceUp) {
		value = m_Rank;
		if (value > 10)
			value = 10;
	}
	return value;
}

void Card::Flip() {
	m_IsFaceUp = !(m_IsFaceUp);
}


class Hand {
public:
	Hand();
	virtual ~Hand();
	void Add(Card* pCard);
	void Clear();
	int GetTotal() const;
protected:
	vector<Card*> m_Cards;
};

Hand::Hand() {
	m_Cards.reserve(7);
}

Hand::~Hand() {
	Clear();
}

void Hand::Add(Card* pCard) {
	m_Cards.push_back(pCard);
}

void Hand::Clear() {
	vector<Card*>::iterator iter;
	for (iter = m_Cards.begin(); iter != m_Cards.end(); ++iter) {
		delete* iter;
		*iter = 0;
	}

	m_Cards.clear();
}

int Hand::GetTotal() const {
	if (m_Cards.empty() || (m_Cards[0])->GetValue() == 0)
		return 0;

	int Total = 0;
	bool haveAce = false;
	vector<Card*>::const_iterator iter;
	for (iter = m_Cards.begin(); iter != m_Cards.end(); ++iter) {
		if ((*iter)->GetValue() == Card::ACE)
			haveAce = true;
		Total += (*iter)->GetValue();
	}

	if (haveAce && Total <= 11) {
		Total += 10;
	}

	return Total;
}


class GenericPlayer : public Hand {
	friend ostream& operator<<(ostream& os, const GenericPlayer& aPlayer);
public:
	GenericPlayer(const string& name = "");
	virtual ~GenericPlayer();
	virtual bool IsHitting() const = 0;
	bool IsBusted() const;
	void Bust() const;
protected:
	string m_Name;
};

GenericPlayer::GenericPlayer(const string& name) :
	m_Name(name) {}

GenericPlayer::~GenericPlayer()
{}

bool GenericPlayer::IsBusted() const {
	return (GetTotal() > 21);
}

void GenericPlayer::Bust() const {
	cout << m_Name << " bust!\n";
}


class Player : public GenericPlayer {
public:
	Player(const string& name = "");
	virtual ~Player();
	virtual bool IsHitting() const;
	void Win() const;
	void Lose() const;
	void Push() const;
};

Player::Player(const string& name) :
	GenericPlayer(name) {}

Player::~Player()
{}

bool Player::IsHitting() const {
	cout << "\n" << m_Name << ", do you want a hit? (y/n): ";
	char answer;
	cin >> answer;
	return (answer == 'y' || answer == 'Y');
}

void Player::Win() const {
	cout << m_Name << " wins\n";
}

void Player::Lose() const {
	cout << m_Name << " loses\n";
}

void Player::Push() const {
	cout << m_Name << " pushes\n";
}


class House : public GenericPlayer {
public:
	House(const string& name = "House");
	virtual ~House();
	virtual bool IsHitting() const;
	void FlipFirstCard();
};

House::House(const string& name) :
	GenericPlayer(name) {}

House::~House()
{}

bool House::IsHitting() const {
	return (GetTotal() <= 16);
}

void House::FlipFirstCard() {
	if (!(m_Cards.empty()))
		m_Cards[0]->Flip();

	else
		cout << "No card to flip\n";
}


class Deck : public Hand {
public:
	Deck();
	virtual ~Deck();
	void Populate();
	void Shuffle();
	void Deal(Hand& aHand);
	void AdditionalCards(GenericPlayer& aPlayer);
};

Deck::Deck() {
	m_Cards.reserve(52);
	Populate();
}

Deck::~Deck()
{}

void Deck::Populate() {
	Clear();
	for (int s = Card::CLUBS; s <= Card::SPADES; ++s) {
		for (int r = Card::ACE; r <= Card::KING; ++r) {
			Add(new Card(static_cast<Card::rank>(r), static_cast<Card::suit>(s)));
		}
	}
}

void Deck::Shuffle() {
	random_shuffle(m_Cards.begin(), m_Cards.end());
}

void Deck::Deal(Hand& aHand) {
	if (!(m_Cards.empty())) {
		aHand.Add(m_Cards.back());
		m_Cards.pop_back();
	}

	else
		cout << "Out of cards. Unable to deal\n";
}

void Deck::AdditionalCards(GenericPlayer& aPlayer) {
	while (aPlayer.IsHitting() && !(aPlayer.IsBusted())) {
		Deal(aPlayer);
		cout << aPlayer << endl;
	}

	if (aPlayer.IsBusted())
		aPlayer.Bust();
}


class Game {
public:
	Game(const vector<string>& names);
	~Game();
	void Play();
private:
	Deck m_Deck;
	House m_House;
	vector<Player> m_Players;
};

Game::Game(const vector<string>& names) {
	vector<string>::const_iterator pName;
	for (pName = names.begin(); pName != names.end(); ++pName) {
		m_Players.push_back(Player(*pName));
	}

	srand(static_cast<unsigned int>(time(0)));
	m_Deck.Populate();
	m_Deck.Shuffle();
}

Game::~Game()
{}

void Game::Play() {
	vector<Player>::iterator pPlayer;
	for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer) {
		m_Deck.Deal(*pPlayer);
		m_Deck.Deal(*pPlayer);
	}
	m_Deck.Deal(m_House);
	m_Deck.Deal(m_House);

	m_House.FlipFirstCard();

	for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer)
		cout << *pPlayer << endl;

	cout << m_House << endl;

	for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer)
		m_Deck.AdditionalCards(*pPlayer);

	m_House.FlipFirstCard();
	cout << m_House << endl;

	m_Deck.AdditionalCards(m_House);

	if (m_House.IsBusted()) {
		for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer) {
			if (!(pPlayer->IsBusted()))
				pPlayer->Win();
		}
	}

	else {
		for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer) {
			if (!(pPlayer->IsBusted())) {
				if (pPlayer->GetTotal() > m_House.GetTotal())
					pPlayer->Win();

				else if (pPlayer->GetTotal() < m_House.GetTotal())
					pPlayer->Lose();

				else
					pPlayer->Push();
			}
		}
	}

	for (pPlayer = m_Players.begin(); pPlayer != m_Players.end(); ++pPlayer)
		pPlayer->Clear();
	m_House.Clear();
}

ostream& operator<<(ostream& os, const Card& aCard);
ostream& operator<<(ostream& os, const GenericPlayer& aPlayer);

int main() {
	cout << "\t\t Welcom to BlackJack!\n\n";
	int answer = 0;
	while (answer < 1 || answer > 7) {
		cout << "How many players? (1-7): ";
		cin >> answer;
	}

	vector<string> names;
	string name;
	for (int i = 0; i < answer; ++i) {
		cout << "Enter player name: ";
		cin >> name;
		names.push_back(name);
	}

	cout << "\n";

	Game aGame(names);

	char again = 'y';

	while (again == 'y' || again == 'Y') {
		aGame.Play();
		cout << "Do you want to play again? (Y/N)\n";
		cin >> again;
	}

	return 0;
}

ostream& operator<<(ostream& os, const Card& aCard) {
	const string RANKS[] = { "0", "A", "2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K" };
	const string SUITS[] = { "c", "d", "h", "s" };

	if (aCard.m_IsFaceUp)
		os << RANKS[aCard.m_Rank] << SUITS[aCard.m_Suit];

	else
		os << "XX";

	return os;
}

ostream& operator<<(ostream& os, const GenericPlayer& aPlayer) {
	os << aPlayer.m_Name << ": ";

	vector<Card*>::const_iterator pCard;
	if (!(aPlayer.m_Cards.empty())) {
		for (pCard = aPlayer.m_Cards.begin(); pCard != aPlayer.m_Cards.end(); ++pCard)
			os << *(*pCard) << "\t";

		if (aPlayer.GetTotal() != 0)
			cout << "(" << aPlayer.GetTotal() << ")";
	}

	else
		os << "<empty>\n";

	return os;
}