import unittest
from unittest.mock import patch
from flask import template_rendered
from app import app, get_exchange_rate

class TestCurrencyConverter(unittest.TestCase):
    def setUp(self):
        app.config['TESTING'] = True
        self.client = app.test_client()
        self.ctx = app.app_context()
        self.ctx.push()

    def tearDown(self):
        self.ctx.pop()

    @patch('app.requests.get')
    def test_get_exchange_rate_success(self, mock_get):
        # Мок успешного ответа API
        mock_response = unittest.mock.Mock()
        mock_response.json.return_value = {'rates': {'EUR': 0.92, 'RUB': 92.0}}
        mock_response.raise_for_status.return_value = None
        mock_get.return_value = mock_response

        rate = get_exchange_rate('USD', 'EUR')
        self.assertEqual(rate, 0.92)

    @patch('app.requests.get')
    def test_get_exchange_rate_failure(self, mock_get):
        # Мок ошибки API
        mock_get.side_effect = Exception("API error")
        rate = get_exchange_rate('USD', 'EUR')
        self.assertIsNone(rate)

    def test_index_get(self):
        response = self.client.get('/')
        self.assertEqual(response.status_code, 200)
        self.assertIn(b'Конвертер валют', response.data)

    @patch('app.get_exchange_rate')
    def test_valid_conversion(self, mock_rate):
        # Тест успешной конвертации
        mock_rate.return_value = 0.92
        response = self.client.post('/', data={
            'amount': '100',
            'from_curr': 'USD',
            'to_curr': 'EUR'
        })
        self.assertIn(b'100 USD = 92.0 EUR', response.data)

    def test_invalid_amount(self):
        # Тест невалидной суммы
        response = self.client.post('/', data={
            'amount': 'invalid',
            'from_curr': 'USD',
            'to_curr': 'EUR'
        })
        self.assertIn(b'Введите корректную сумму', response.data)

    @patch('app.get_exchange_rate')
    def test_invalid_currency(self, mock_rate):
        # Тест несуществующей валюты
        mock_rate.return_value = None
        response = self.client.post('/', data={
            'amount': '100',
            'from_curr': 'AAA',
            'to_curr': 'BBB'
        })
        self.assertIn(b'Ошибка конвертации', response.data)

    def test_missing_fields(self):
        # Тест пропущенных полей
        response = self.client.post('/', data={'amount': '100'})
        self.assertIn(b'Выберите валюты', response.data)

    @patch('app.get_exchange_rate')
    def test_edge_cases(self, mock_rate):
        # Тест граничных значений
        mock_rate.return_value = 1.0
        test_cases = [
            ('0', 'USD', 'USD', b'0 USD = 0.0 USD'),
            ('0.01', 'EUR', 'RUB', b'0.01 EUR = 0.01 RUB'),
            ('1000000', 'JPY', 'CNY', b'1000000 JPY = 1000000.0 CNY')
        ]

        for amount, from_curr, to_curr, expected in test_cases:
            with self.subTest(f"{amount} {from_curr}->{to_curr}"):
                response = self.client.post('/', data={
                    'amount': amount,
                    'from_curr': from_curr,
                    'to_curr': to_curr
                })
                self.assertIn(expected, response.data)

if __name__ == '__main__':
    unittest.main()
